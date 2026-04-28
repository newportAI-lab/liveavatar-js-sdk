[SDK用户手册](./用户使用手册)|[SDK_User_Manual](./SDK_User_Manual.md)
# 🤖 Real-time Audio-Video Interaction SDK User Guide

This SDK provides an integrated communication solution combining **LiveKit (RTC)** , specifically designed for digital human conversations, real-time voice interactions, and similar scenarios. It handles complex underlying logic such as media track synchronization, audio capture, streaming text parsing, and automatic reconnection.

## Performance Monitoring

The SDK includes a built-in low-intrusion performance monitor to simplify integration with observability systems (SkyWalking/OpenTelemetry/custom reporters).

```ts
import { createClient, type PerformanceMetricRecord } from '@sanseng/livekit-ws-sdk';

const client = createClient({
  connectConfig: {
    type: 'direct',
    config: {
      sfuUrl: '',
      clientToken: '',
    },
  },
});
```
