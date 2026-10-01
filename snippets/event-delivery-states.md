# Event delivery states

| State | Meaning | Recommended action |
| --- | --- | --- |
| `pending` | The event has been queued but delivery has not started. | No action required. |
| `delivering` | Orbit is attempting delivery. | No action required unless the state persists. |
| `delivered` | The endpoint returned a successful 2xx response. | Confirm downstream processing if needed. |
| `retrying` | A previous delivery attempt failed and Orbit will retry. | Review endpoint availability and recent response codes. |
| `failed` | The retry window ended without a successful response. | Resolve the endpoint issue, then replay the event. |
| `disabled` | Delivery was paused after repeated failures or by an administrator. | Correct the cause and re-enable the subscription. |
