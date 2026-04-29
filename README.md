# Facemarket Live Avatar JS SDK User Guide

This SDK provides an integrated communication solution combining **LiveKit (RTC)** , specifically designed for live avatar conversations, real-time voice interactions, and similar scenarios. It handles complex underlying logic such as media track synchronization, audio capture, streaming text parsing, and automatic reconnection.

## Documentation

For detailed integration guides and API references, please refer to the:

* [SDK User Manual (English)](./SDK_User_Manual.md)
* [SDK 用户手册 (中文)](./SDK_使用手册.md)

## Performance Monitoring

The SDK includes a built-in low-intrusion performance monitor to simplify integration with observability systems (SkyWalking/OpenTelemetry/custom reporters).

```ts
import { createClient, type PerformanceMetricRecord } from '@sanseng/liveavatar-js-sdk';

const client = createClient({
  connectConfig: {
    type: 'direct',
    config: {
      sfuUrl: '',
      userToken: '',
    },
  },
});
```
