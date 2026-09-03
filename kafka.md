# Lab: Kafka Alert Destination + Event Correlation in Cribl Stream (On-Prem)

**Environment assumed:** 1 Cribl Stream Leader + 1 Worker, both on Rocky Linux
**Kafka:** deployed locally via Docker on the same host (or a reachable host) — no existing Kafka cluster required

---

## Part A: Deploy Kafka via Docker

### A1. Install Docker (if not already installed)
```bash
sudo dnf install -y docker
sudo systemctl enable --now docker
```

### A2. Install Docker Compose plugin (if needed)
```bash
sudo dnf install -y docker-compose-plugin
```

### A3. Deploy Kafka
Use the provided `docker-compose-kafka.yml`. Before running it, replace `<HOST_IP_OR_HOSTNAME>` with the actual IP or hostname of the machine running Kafka (this is what Cribl Workers will connect to).

```bash
docker compose -f docker-compose-kafka.yml up -d
```

This starts a single-broker Kafka instance in KRaft mode (no separate Zookeeper container needed), listening on:
- `9092` — internal/container-network clients
- `9094` — external listener, mapped to the host, for Cribl to connect to

Check it started cleanly:
```bash
docker logs -f kafka-lab
```
Look for a line indicating the broker has started, then `Ctrl+C`.

### A4. Open the Firewall (Rocky Linux uses firewalld by default)
```bash
sudo firewall-cmd --permanent --add-port=9094/tcp
sudo firewall-cmd --reload
```

### A5. Create the Topic
```bash
docker exec -it kafka-lab kafka-topics.sh \
  --create --topic cribl-alerts \
  --bootstrap-server localhost:9092 \
  --partitions 1 --replication-factor 1
```

Verify it exists:
```bash
docker exec -it kafka-lab kafka-topics.sh \
  --list --bootstrap-server localhost:9092
```

---

## Part A2: Confirm/Create the Input Source

This lab assumes you're reusing the **HTTP Source on port 10080** created in the earlier lookup enrichment lab. If that's still configured, skip to Part B.

**If you don't already have it, create it now:**
1. Go to **Data → Sources → Add Source → HTTP/S (Raw)**
2. **Input ID:** `lab-http-source`
3. **Address:** `0.0.0.0` (listen on all interfaces) or `localhost` if testing only from the same host
4. **Port:** `10080`
5. Toggle **Enabled** on
6. Click **Save**
7. **Commit and Deploy**

You'll route this Source's data through the pipeline described in Part C, ending at the Kafka Destination from Part B — see Part D for the Route wiring.

---

## Part B: Configure the Kafka Destination in Cribl Stream

1. Log into the Cribl Stream Leader UI
2. Select the relevant **Worker Group**
3. Go to **Data → Destinations → Add Destination → Kafka**
4. Configure **General Settings**:
   - **Output ID:** `kafka-alerts`
   - **Bootstrap servers:** `<HOST_IP_OR_HOSTNAME>:9094`
   - **Topic:** `cribl-alerts`
5. **Optional Settings:**
   - **Record data format:** JSON
   - **Compression:** Gzip (Cribl recommends enabling compression)
   - **Acknowledgments:** Leader (default) is fine for a lab; use `All` in production for stronger durability guarantees
6. Since this is a local, unauthenticated Docker broker, leave **Authentication** set to none and **TLS** disabled. (If you later point this at a real cluster using SASL or mTLS, see the Facilitator Notes at the end.)
7. Click **Save**
8. Click **Commit and Deploy** (distributed deployments require this before changes take effect on Workers)

---

## Part C: Event Correlation Setup

Cribl Stream doesn't have a dedicated "correlation engine," but the **Aggregations Function** is the standard building block for correlating related events over a time window — e.g., counting related events per host and emitting a single summary/alert event when a threshold is crossed, instead of forwarding every raw event.

### C1. Add the Aggregations Function to Your Pipeline
Using the `lookup-enrichment-pipeline` from the earlier lab (or a new pipeline):

1. Open the pipeline, click **Add Function**, search for **Aggregations**
2. Configure it:
   - **Group by:** `host` (correlate events per host — swap in `department` or another field depending on what you want to correlate on)
   - **Time window:** `60s` (adjust based on how "related" events need to be in time to count as correlated)
   - **Aggregations:**
     - `count()` → output field `event_count`
     - `first(department)` → output field `department`
     - `first(criticality)` → output field `criticality`
     - `values(host)` → output field `matched_hosts` (optional, if grouping on a broader field)
3. Click **Save**

### C2. Add a Threshold Check (Turns Aggregation into an "Alert")
Add an **Eval** Function after Aggregations:
```
is_alert = event_count >= 3
```
Adjust the threshold (`3`) to whatever makes sense for your correlation logic — e.g., "3 or more events from the same host within 60 seconds."

### C3. Filter So Only Alerts Go to Kafka
Add a **Drop** Function (or use a Route filter instead — see Part D) configured with:
```
Filter: is_alert !== true
```
This drops everything that isn't a correlated alert, so only real alert events reach Kafka. (Alternative: skip this Function and instead filter directly in the Route, shown below.)

---

## Part D: Route Correlated Alerts to Kafka

1. Go to **Routing → Routes** (or **QuickConnect**)
2. Add a new Route:
   - **Filter:** `is_alert === true` (if you didn't already drop non-alerts in the pipeline)
   - **Pipeline:** the pipeline containing your Aggregations + Eval Functions
   - **Destination:** `kafka-alerts`
3. Order this Route above your existing Devnull/testing route if needed, so alert traffic doesn't get swallowed by a catch-all
4. **Commit and Deploy**

---

## Part E: End-to-End Validation & Testing

### E1. Start a Kafka Consumer to Watch for Messages
In a separate terminal:
```bash
docker exec -it kafka-lab kafka-console-consumer.sh \
  --topic cribl-alerts \
  --bootstrap-server localhost:9092 \
  --from-beginning
```
Leave this running — any message Cribl sends to the topic will print here in real time.

### E2. Generate Enough Correlated Events to Trigger the Alert
Since the threshold is `event_count >= 3`, send the same host multiple times within the 60-second window. For example, repeat `worker-10.0.5.2` three times:

```bash
for i in 1 2 3; do
  curl -s -X POST "http://localhost:10080/" \
       -H "Content-Type: application/json" \
       -d '{"_raw":"test log line from rocky linux","_time":'"$(date +%s)"',"host":"worker-10.0.5.2","index":"default","source":"http:cribl"}'
  echo
  sleep 2
done
```

### E3. Confirm the Alert Appears
- In the Kafka consumer terminal (E1), you should see one JSON message appear — the correlated alert — after the aggregation window closes
- In Cribl, go to **Monitoring → Data → Live Data** on the `kafka-alerts` Destination and confirm the event shows `event_count: 3`, `is_alert: true`, plus the enriched `department`/`criticality`/`location` fields from the earlier lookup step

### E4. Negative Test
Send only 1–2 events for a different host (e.g., `worker-10.0.5.17`) and confirm **no** message appears in the Kafka consumer — proving the threshold logic correctly suppresses non-correlated, low-volume traffic.

### E5. Confirm Delivery Guarantees (Optional, Production-Mindset Check)
- Stop the `kafka-lab` container mid-test and confirm Cribl's Persistent Queue buffers events (check **Monitoring → Destinations → kafka-alerts** for PQ activity) rather than silently dropping them
- Restart the container and confirm buffered events flush through once the broker is back

---

## Facilitator Notes
- This lab uses a single unauthenticated Kafka broker for simplicity. If your real Kafka cluster requires SASL/SCRAM or mTLS, the Kafka Destination in Cribl exposes **Authentication** and **TLS Settings** sections — swap in your real bootstrap servers, credentials, and certificates there instead of the local Docker broker.
- The Aggregations Function is a light-weight correlation tool — for complex multi-event-type correlation (e.g., matching a login event to a later privilege-escalation event across different sources), consider whether a downstream SIEM/SOAR correlation engine is more appropriate; Cribl's role is often to reduce and reshape data before it reaches that layer.
- Remember: **any config change on a distributed (Leader/Worker) deployment requires Commit and Deploy** before Workers pick it up.
