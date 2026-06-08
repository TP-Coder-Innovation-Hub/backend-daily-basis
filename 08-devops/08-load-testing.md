# Load Testing — k6 for Spring Boot APIs

## Why Load Test

Unit tests verify correctness. Load tests verify your service handles real traffic. Without load testing, you discover performance problems in production.

## Step 1: Install k6

```bash
brew install k6
```

## Step 2: Basic Load Test Script

```javascript
// load-tests/orders.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

const errorRate = new Rate('errors');
const latency = new Trend('order_latency');

export const options = {
  stages: [
    { duration: '30s', target: 20 },
    { duration: '1m', target: 50 },
    { duration: '30s', target: 0 },
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],
    errors: ['rate<0.05'],
    http_req_failed: ['rate<0.01'],
  },
};

const BASE_URL = 'http://localhost:8080';

export function setup() {
  const loginRes = http.post(`${BASE_URL}/api/auth/login`,
    JSON.stringify({ username: 'test', password: 'test' }),
    { headers: { 'Content-Type': 'application/json' } });
  return { token: loginRes.json('accessToken') };
}

export default function (data) {
  const params = {
    headers: {
      'Authorization': `Bearer ${data.token}`,
      'Content-Type': 'application/json',
    },
  };

  const res = http.get(`${BASE_URL}/api/orders?page=0&size=20`, params);

  check(res, {
    'status is 200': (r) => r.status === 200,
    'has content': (r) => r.json().content.length > 0,
  });

  errorRate.add(res.status !== 200);
  latency.add(res.timings.duration);

  sleep(1);
}

export function teardown(data) {
}
```

## Step 3: Run and Analyze

```bash
k6 run load-tests/orders.js
```

Output:
```
scenarios: (100.00%) 1 scenario, 50 max VUs
  default: Up to 50 looping VUs for 2m0s

     ✓ status is 200
     ✓ has content

checks.........................: 100.00% ✓ 1234
data_received..................: 4.5 MB
data_sent......................: 1.2 MB
http_req_duration..............: avg=125ms  p(95)=340ms
http_req_failed................: 0.00%
http_reqs......................: 5678
iterations.....................: 5678
vus............................: 50
```

## Step 4: Test Scenarios

```javascript
// Ramp-up stress test
export const options = {
  stages: [
    { duration: '2m', target: 100 },
    { duration: '5m', target: 100 },
    { duration: '2m', target: 200 },
    { duration: '5m', target: 200 },
    { duration: '2m', target: 0 },
  ],
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'],
    errors: ['rate<0.01'],
  },
};
```

```javascript
// Spike test — sudden traffic burst
export const options = {
  stages: [
    { duration: '10s', target: 100 },
    { duration: '1m', target: 100 },
    { duration: '10s', target: 500 },
    { duration: '1m', target: 500 },
    { duration: '10s', target: 100 },
    { duration: '1m', target: 100 },
    { duration: '10s', target: 0 },
  ],
};
```

## Step 5: CI Integration

```yaml
# .github/workflows/load-test.yml
name: Load Test
on:
  schedule:
    - cron: '0 2 * * *'

jobs:
  load-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: grafana/setup-k6-action@v1
      - name: Run load tests
        run: k6 run load-tests/orders.js
        env:
          K6_CLOUD_TOKEN: ${{ secrets.K6_TOKEN }}
```

## Thresholds for Pass/Fail

| Metric | Threshold | Meaning |
|--------|-----------|---------|
| `p(95) < 500ms` | 95% of requests under 500ms | Acceptable latency |
| `p(99) < 1000ms` | 99% of requests under 1s | Tail latency bound |
| `errors rate < 0.01` | Less than 1% error rate | Reliability target |
| `http_req_failed rate < 0.01` | Less than 1% HTTP failures | Connection reliability |

If any threshold is breached, k6 exits with a non-zero code — the CI build fails.

## Key Points

- Load test before release, not in production
- Define thresholds based on your SLOs — the test fails if you break your promises
- Use stages to simulate realistic traffic patterns (ramp-up, steady, spike)
- Authenticate once in `setup()`, reuse the token across all VUs
- Run load tests in CI on a schedule (nightly) to catch regressions
