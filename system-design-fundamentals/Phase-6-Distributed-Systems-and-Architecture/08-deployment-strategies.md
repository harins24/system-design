# Deployment Strategies

**Phase 6 — Distributed Systems + Architecture | Topic 8**

---

## Why Deployment Strategy Matters

Deploying software is one of the highest-risk moments in a system's life. Every deployment is a potential outage.

```
Naive deployment (stop, replace, start):
  t=0:   Stop old version → users get errors
  t=30s: New code deployed
  t=60s: New code starts up
  t=90s: New code healthy → users served again

  Result: 90 seconds of downtime per deployment
  At 10 deploys/day: 15 minutes downtime daily
  At 100 services: catastrophic

  Also: new version has bug → entire user base affected immediately
        No easy rollback (restarting old version is itself risky)

Modern deployment strategies:
  Zero downtime (no gap between old and new)
  Gradual rollout (limit blast radius of bugs)
  Instant rollback (revert in seconds)
  Test in production (validate before full rollout)
```

---

## Strategy 1: Rolling Deployment

Replace instances of the old version with the new version gradually, one at a time (or in batches).

```
10 instances running v1.0

Step 1:  Replace instance 1 with v2.0
         9 instances running v1.0 + 1 running v2.0

Step 2:  Replace instances 2-3 with v2.0
         7 instances running v1.0 + 3 running v2.0

Step 3:  Replace instances 4-7 with v2.0
         3 instances running v1.0 + 7 running v2.0

Step 4:  Replace instances 8-10 with v2.0
         0 instances running v1.0 + 10 running v2.0
         Complete!
```

### Rolling Deployment in Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2        # Allow 2 extra pods during update (12 total max)
      maxUnavailable: 1  # Only 1 pod down at a time (9 minimum)
  template:
    spec:
      containers:
      - name: order-service
        image: order-service:v2.0
        readinessProbe:           # Only add to LB when healthy
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
```

```
Rolling update process:
  kubectl set image deployment/order-service \
    order-service=order-service:v2.0

Kubernetes:
  Creates new v2.0 pod (11 total)
  Waits for v2.0 pod to pass readiness probe
  Removes one v1.0 pod from load balancer (10 total)
  Terminates v1.0 pod after graceful shutdown
  Repeats until all replaced

Zero downtime: always at least 9 pods serving traffic
Blast radius: ~10% of traffic hits new version initially
```

**Pros/Cons:**

```
✅ Zero downtime
✅ Gradual rollout (small % affected if bug in new version)
✅ Simple to implement (Kubernetes default)
✅ Automatic rollback if pods fail readiness checks

❌ Slow (deploying 100 instances one at a time takes time)
❌ Old and new versions run simultaneously (API compatibility required)
❌ No instant traffic control (can't quickly rollback to 100% old)
❌ Hard to test the new version before it handles production traffic
```

---

## Strategy 2: Blue-Green Deployment

Maintain two identical environments. Switch traffic between them instantly.

```
Current state:
  Blue environment:  v1.0 running (serving 100% of traffic)
  Green environment: v1.0 running (idle standby)

Deploy v2.0:
  Step 1: Deploy v2.0 to GREEN environment (still idle)
          Blue: v1.0 serving 100% ✅
          Green: v2.0 deployed, run smoke tests ← not live yet

  Step 2: All tests pass → switch load balancer
          Blue: v1.0 idle (standby)
          Green: v2.0 serving 100% ✅

  Instant rollback:
          Bug detected → switch LB back to Blue
          Blue: v1.0 serving 100% ✅ (within seconds)
          Green: v2.0 idle

  After confidence period:
          Green becomes new standby
          Blue updated to v2.0 for next deploy
```

### AWS Implementation

```yaml
# AWS CodeDeploy Blue-Green with ECS

DeploymentGroup:
  DeploymentType: BLUE_GREEN
  BlueGreenDeploymentConfiguration:
    DeploymentReadyOption:
      ActionOnTimeout: CONTINUE_DEPLOYMENT  # auto-switch after tests
      WaitTimeInMinutes: 10                  # or manual approval
    GreenFleetProvisioningOption:
      Action: DISCOVER_EXISTING             # use existing green env
    TerminateBlueInstancesOnDeploymentSuccess:
      Action: TERMINATE
      TerminationWaitTimeInMinutes: 60      # keep blue alive 60min

# Route 53 traffic switching
aws route53 change-resource-record-sets \
  --hosted-zone-id ZONE123 \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "A",
        "AliasTarget": {
          "DNSName": "green-alb.amazonaws.com"  # switch to green
        }
      }
    }]
  }'
```

**Pros/Cons:**

```
✅ Instant rollback (flip LB back — seconds)
✅ Test new version thoroughly before going live
✅ No mixed versions running simultaneously
✅ Zero downtime

❌ 2x infrastructure cost (two full environments)
❌ Database migrations are hard (both versions must work with same DB)
❌ Long-lived connections not automatically migrated
❌ Expensive for large systems
```

### Database Migration Problem

```
Blue-Green with DB migrations:

v1.0 schema: users table (id, name, email)
v2.0 schema: users table (id, first_name, last_name, email)
             (split name into first_name + last_name)

Problem:
  Blue (v1.0) uses "name" column
  Green (v2.0) uses "first_name" + "last_name"
  Same database!

  Switch to Green → v1.0 can't work if "name" column removed
  Keep both → complex migration

Solution — Expand-Contract Pattern:
  Phase 1 (Expand): Add new columns WITHOUT removing old
    ALTER TABLE users ADD COLUMN first_name VARCHAR(50);
    ALTER TABLE users ADD COLUMN last_name VARCHAR(50);
    -- Both old and new columns exist, both versions work

  Phase 2: Deploy v2.0 — writes both old and new columns
    "Hari Kumar" → name="Hari Kumar",
                   first_name="Hari", last_name="Kumar"

  Phase 3: All traffic on v2.0, data migrated

  Phase 4 (Contract): Remove old column
    ALTER TABLE users DROP COLUMN name;
    -- Safe now — only v2.0 running, uses only new columns

  Takes multiple deployment cycles
  Never break existing running version
```

---

## Strategy 3: Canary Deployment

Route a small percentage of traffic to the new version. Monitor closely. Gradually increase if healthy.

```
Start:
  v1.0: 100% of traffic

Step 1 (Canary):
  v1.0: 95% of traffic
  v2.0: 5% of traffic   ← canary
  Monitor: errors, latency, business metrics

Step 2 (Expand if healthy):
  v1.0: 75%
  v2.0: 25%
  Monitor 30 minutes

Step 3:
  v1.0: 50%
  v2.0: 50%
  Monitor 30 minutes

Step 4:
  v1.0: 10%
  v2.0: 90%

Step 5 (Full rollout):
  v1.0: 0%
  v2.0: 100%

Rollback at any step:
  Route all traffic back to v1.0
  5% of users affected, not 100%
```

### Canary with Istio

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
  - order-service
  http:
  - route:
    - destination:
        host: order-service
        subset: stable    # v1.0
      weight: 95
    - destination:
        host: order-service
        subset: canary    # v2.0
      weight: 5
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: order-service
spec:
  host: order-service
  subsets:
  - name: stable
    labels:
      version: v1.0
  - name: canary
    labels:
      version: v2.0
```

### Automated Canary Analysis (Argo Rollouts)

```yaml
# Automatically promote or rollback based on metrics
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: order-service
spec:
  strategy:
    canary:
      steps:
      - setWeight: 5      # 5% canary
      - analysis:         # analyze for 5 minutes
          templates:
          - templateName: success-rate
          args:
          - name: service-name
            value: order-service
      - setWeight: 25     # 25% if analysis passed
      - analysis:
          templates:
          - templateName: success-rate
          args:
          - name: service-name
            value: order-service
      - setWeight: 50
      - setWeight: 100    # full rollout
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
  - name: success-rate
    successCondition: result[0] >= 0.99  # 99% success required
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          sum(rate(http_requests_total{service="{{args.service-name}}",
                   status!~"5.."}[5m])) /
          sum(rate(http_requests_total{service="{{args.service-name}}"}[5m]))
```

**Pros/Cons:**

```
✅ Real production validation before full rollout
✅ Small blast radius for bugs (5% initially)
✅ Gradual rollout builds confidence
✅ Automatic rollback based on metrics

❌ Slow (hours to fully deploy with monitoring steps)
❌ Complex traffic splitting infrastructure needed
❌ Both versions run simultaneously (API compatibility)
❌ Harder to debug issues split across two versions
```

---

## Strategy 4: Feature Flags (Feature Toggles)

Deploy code to all servers but control activation through configuration flags.

```
v2.0 deployed to all servers, but new feature gated:

if (featureFlags.isEnabled("new-checkout-flow", userId)) {
    return newCheckoutFlow.process(request);
} else {
    return oldCheckoutFlow.process(request);
}

Feature flag states:
  Off:          0% of users see new feature
  Internal:     Only employees see new feature (testing)
  5% rollout:   Random 5% of users
  Power users:  Opt-in users testing new feature
  100%:         All users
  Off:          Rolled back (flag turned off, no redeployment!)
```

### Feature Flag Implementation

```java
@Service
public class CheckoutService {

    private final FeatureFlagService flags;
    private final NewCheckoutFlow newFlow;
    private final OldCheckoutFlow oldFlow;

    public CheckoutResult checkout(CheckoutRequest request) {
        if (flags.isEnabled("new-checkout-flow", request.getUserId())) {
            return newFlow.process(request);
        }
        return oldFlow.process(request);
    }
}

// Feature flag service (uses LaunchDarkly / Unleash / custom)
@Service
public class FeatureFlagService {

    private final LDClient ldClient;  // LaunchDarkly

    public boolean isEnabled(String flagKey, String userId) {
        LDUser user = new LDUser.Builder(userId).build();
        return ldClient.boolVariation(flagKey, user, false);
    }

    public <T> T getVariant(String flagKey, String userId, T defaultValue) {
        LDUser user = new LDUser.Builder(userId).build();
        return ldClient.variation(flagKey, user, defaultValue);
    }
}
```

### A/B Testing with Feature Flags

```java
// A/B test: which checkout flow converts better?
public CheckoutResult checkout(CheckoutRequest request) {
    String variant = flags.getVariant(
        "checkout-flow-experiment",
        request.getUserId(),
        "control"  // default
    );

    CheckoutResult result;
    switch (variant) {
        case "treatment-a":
            result = onePageCheckout.process(request);
            break;
        case "treatment-b":
            result = expressCheckout.process(request);
            break;
        default: // "control"
            result = standardCheckout.process(request);
    }

    // Track experiment data
    analytics.track("checkout.completed", Map.of(
        "variant", variant,
        "userId", request.getUserId(),
        "total", result.getTotal()
    ));

    return result;
}
```

**Pros/Cons:**

```
✅ Instant rollback (turn off flag — no deployment)
✅ Dark launches (code in production, feature off)
✅ Per-user targeting (employees first, power users, gradual)
✅ A/B testing built in
✅ Separate deployment from release

❌ Technical debt: old code paths accumulate
❌ Flag cleanup required (remove after full rollout)
❌ Testing complexity: must test all flag combinations
❌ Flag sprawl: too many flags = confusion

Best practice:
  Short-lived flags: remove within 1-2 sprints after full rollout
  Long-lived flags: operational flags (kill switches, config)
  Flag inventory: track purpose, owner, expiry date
```

---

## Strategy 5: Shadow Deployment

Send duplicate traffic to the new version without affecting users.

```
Production traffic:
  Request → v1.0 (serves real response to user)
           → v2.0 (receives same request, processes it,
                   response DISCARDED — not sent to user)

  Compare:
  v1.0 response: {total: 99.99, tax: 8.00}
  v2.0 response: {total: 101.49, tax: 9.50}

  Difference detected! v2.0 has different tax calculation.
  Fix v2.0 before switching traffic.

Use cases:
  Validate new ML model on real traffic
  Test new database query behavior on production load
  Verify new service version handles real request shapes
  Regression testing at production scale
```

```yaml
# Istio shadow traffic
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
spec:
  http:
  - route:
    - destination:
        host: order-service-v1
    mirror:                          # shadow copy
      host: order-service-v2
    mirrorPercentage:
      value: 100.0                   # 100% of traffic mirrored
```

---

## Graceful Shutdown — Critical for All Strategies

During any deployment, old pods are terminated. They must finish in-flight requests first.

```java
@SpringBootApplication
public class OrderServiceApplication {

    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(OrderServiceApplication.class);
        app.run(args);
    }
}
```

```yaml
# application.yml
server:
  shutdown: graceful           # enable graceful shutdown

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s  # wait up to 30s for in-flight requests

# Kubernetes preStop hook gives time before SIGTERM
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 10"]
# Why: kube-proxy takes time to remove pod from service endpoints
# Without sleep: new requests arrive in the gap between
#                pod removed from Kubernetes service and SIGTERM sent
```

---

## Choosing a Deployment Strategy

```
Rolling:
  Use when: standard deployments, Kubernetes-native simplicity
  When: most microservice updates
  Limitation: both versions run together, slow for many instances

Blue-Green:
  Use when: instant rollback critical, high-risk changes
  When: major releases, schema migrations, high-traffic events
  Limitation: 2x infrastructure cost

Canary:
  Use when: user-facing changes, uncertainty about impact
  When: new features, performance changes, behavioral changes
  Limitation: slow, complex infrastructure

Feature Flags:
  Use when: release separate from deployment desired
  When: A/B testing, gradual user rollout, kill switches
  Limitation: code cleanup debt

Shadow:
  Use when: must validate new version on real traffic safely
  When: ML model rollout, major algorithm changes
  Limitation: double infrastructure cost, complex comparison
```

---

## Key Takeaways

```
Deployment strategy: how you replace running software
Goal: zero downtime + limited blast radius + fast rollback

Rolling:
  Replace instances gradually, one batch at a time
  Kubernetes default (RollingUpdate)
  maxSurge + maxUnavailable control pace
  Zero downtime, but both versions coexist

Blue-Green:
  Two identical environments, switch LB instantly
  Zero downtime, instant rollback
  Cost: 2x infrastructure
  Challenge: database migrations (Expand-Contract pattern)

Canary:
  Small % traffic to new version, expand if healthy
  Istio / Argo Rollouts for traffic splitting
  Automated analysis (Prometheus metrics) for auto-promote/rollback
  Small blast radius: 5% affected, not 100%

Feature Flags:
  Deploy code, control activation via config
  Instant rollback without redeployment
  Enable: dark launch, A/B testing, per-user rollout
  Discipline required: remove old flags promptly

Shadow:
  Mirror traffic to new version, discard response
  Validate behavior before any users affected
  Good for: ML models, major algorithm changes

Graceful shutdown:
  Critical for all strategies
  spring.lifecycle.timeout-per-shutdown-phase: 30s
  preStop hook: sleep before SIGTERM for kube-proxy sync

Expand-Contract for DB migrations:
  Never break running version
  Add columns first, migrate data, remove old column later
  Multiple deployment cycles per schema change
```
