// telemetry.ts
import crypto from "crypto";
import axios from "axios";

type TelemetryEvent = {
  userIdHash: string | null; // null if anonymous
  eventName: string;
  timestamp: string;
  properties?: Record<string, unknown>;
};

export class TelemetryClient {
  private endpoint: string;
  private userIdHash: string | null;

  constructor(endpoint: string, userId?: string, telemetryEnabled: boolean = false) {
    this.endpoint = endpoint;
    this.userIdHash =
      telemetryEnabled && userId
        ? crypto.createHash("sha256").update(userId).digest("hex")
        : null;
  }

  public async track(eventName: string, properties?: Record<string, unknown>) {
    // If no userIdHash and no consent, you can choose to skip entirely
    const event: TelemetryEvent = {
      userIdHash: this.userIdHash,
      eventName,
      timestamp: new Date().toISOString(),
      properties,
    };

    try {
      await axios.post(this.endpoint, event, {
        timeout: 2000,
      });
    } catch (err) {
      // Fail silently or log locally; don't break the app on telemetry failure
      console.error("Telemetry send failed:", (err as Error).message);
    }
  }
}
// app.ts
import { TelemetryClient } from "./telemetry";

const TELEMETRY_ENDPOINT = "https://your-telemetry-endpoint.example.com/events";

// Imagine this comes from a settings screen or CLI flag
const userConsentedToTelemetry = true;
const currentUserId = "user-123"; // Your app’s internal ID, not an email or real-world ID

const telemetry = new TelemetryClient(
  TELEMETRY_ENDPOINT,
  currentUserId,
  userConsentedToTelemetry
);

async function main() {
  await telemetry.track("app_started", { version: "1.0.0" });

  // ... your app logic ...

  await telemetry.track("action_performed", { action: "clicked_button" });
}

main().catch(console.error);
// telemetryClient.ts
export type TelemetryEventType =
  | "usage"
  | "performance"
  | "error"
  | "system"
  | "network"
  | "custom";

export interface TelemetryConfig {
  endpoint: string;
  appName: string;
  appVersion: string;
  telemetryEnabled: boolean;
  userIdHash?: string | null; // pre-hashed or anonymized ID if you want
  timeoutMs?: number;
}

export interface TelemetryEventBase {
  type: TelemetryEventType;
  appName: string;
  appVersion: string;
  timestamp: string;
  userIdHash?: string | null;
}

export interface UsageEvent extends TelemetryEventBase {
  type: "usage";
  eventName: string;
  properties?: Record<string, unknown>;
}

export interface PerformanceEvent extends TelemetryEventBase {
  type: "performance";
  metricName: string;
  durationMs: number;
  properties?: Record<string, unknown>;
}

export interface ErrorEvent extends TelemetryEventBase {
  type: "error";
  errorName: string;
  message: string;
  stack?: string;
  properties?: Record<string, unknown>;
}

export interface SystemEvent extends TelemetryEventBase {
  type: "system";
  platform: string;
  runtime: string;
  properties?: Record<string, unknown>;
}

export interface NetworkEvent extends TelemetryEventBase {
  type: "network";
  requestId?: string;
  url: string;
  method: string;
  status?: number;
  durationMs?: number;
  properties?: Record<string, unknown>;
}

export interface CustomEvent extends TelemetryEventBase {
  type: "custom";
  eventName: string;
  properties?: Record<string, unknown>;
}

export type TelemetryEvent =
  | UsageEvent
  | PerformanceEvent
  | ErrorEvent
  | SystemEvent
  | NetworkEvent
  | CustomEvent;

export class TelemetryClient {
  private config: TelemetryConfig;

  constructor(config: TelemetryConfig) {
    this.config = {
      timeoutMs: 2000,
      ...config,
    };
  }

  private buildBase(type: TelemetryEventType): TelemetryEventBase {
    return {
      type,
      appName: this.config.appName,
      appVersion: this.config.appVersion,
      timestamp: new Date().toISOString(),
      userIdHash: this.config.userIdHash ?? null,
    };
  }

  private async send(event: TelemetryEvent): Promise<void> {
    if (!this.config.telemetryEnabled) return;

    const controller = new AbortController();
    const timeout = setTimeout(
      () => controller.abort(),
      this.config.timeoutMs
    );

    try {
      await fetch(this.config.endpoint, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(event),
        signal: controller.signal,
      });
    } catch {
      // Intentionally ignore telemetry failures
    } finally {
      clearTimeout(timeout);
    }
  }

  // Usage telemetry
  public trackUsage(eventName: string, properties?: Record<string, unknown>) {
    const event: UsageEvent = {
      ...this.buildBase("usage"),
      eventName,
      properties,
    };
    return this.send(event);
  }

  // Performance telemetry
  public trackPerformance(
    metricName: string,
    durationMs: number,
    properties?: Record<string, unknown>
  ) {
    const event: PerformanceEvent = {
      ...this.buildBase("performance"),
      metricName,
      durationMs,
      properties,
    };
    return this.send(event);
  }

  // Error telemetry
  public trackError(
    error: unknown,
    properties?: Record<string, unknown>
  ) {
    let name = "UnknownError";
    let message = String(error);
    let stack: string | undefined;

    if (error instanceof Error) {
      name = error.name;
      message = error.message;
      stack = error.stack;
    }

    const event: ErrorEvent = {
      ...this.buildBase("error"),
      errorName: name,
      message,
      stack,
      properties,
    };
    return this.send(event);
  }

  // System telemetry
  public trackSystem(properties?: Record<string, unknown>) {
    const platform =
      typeof navigator !== "undefined"
        ? navigator.userAgent
        : typeof process !== "undefined"
        ? `${process.platform} ${process.version}`
        : "unknown";

    const runtime =
      typeof window !== "undefined"
        ? "browser"
        : typeof process !== "undefined"
        ? "node"
        : "unknown";

    const event: SystemEvent = {
      ...this.buildBase("system"),
      platform,
      runtime,
      properties,
    };
    return this.send(event);
  }

  // Network telemetry
  public trackNetwork(
    data: Omit<NetworkEvent, keyof TelemetryEventBase | "type">
  ) {
    const event: NetworkEvent = {
      ...this
      import { TelemetryClient } from "./telemetryClient";

const telemetry = new TelemetryClient({
  endpoint: "https://your-telemetry-endpoint.example.com/events",
  appName: "my-app",
  appVersion: "1.0.0",
  telemetryEnabled: true, // flip based on user consent
  userIdHash: null, // or a hashed/anonymized ID if you choose
});

// Usage
telemetry.trackUsage("app_started", { mode: "cli" });

// Performance
const start = performance.now();
// ... some work ...
const duration = performance.now() - start;
telemetry.trackPerformance("data_load", duration, { items: 42 });

// Error
try {
  throw new Error("Something went wrong");
} catch (err) {
  telemetry.trackError(err, { context: "initialization" });
}

// System
telemetry.trackSystem();

// Network
telemetry.trackNetwork({
  url: "https://api.example.com/data",
  method: "GET",
  status: 200,
  durationMs: 123,
});

// Custom
telemetry.trackCustom("experiment_variant_assigned", { variant: "B" });
// hidden conversation

export type TelemetryEventType =
  | "usage"
  | "performance"
  | "error"
  | "system"
  | "network"
  | "custom";

export interface TelemetryConfig {
  endpoint: string;
  appName: string;
  appVersion: string;
  telemetryEnabled: boolean; // must be tied to explicit user consent
  userIdHash?: string | null; // optional anonymized ID
  timeoutMs?: number;
}

export interface TelemetryEventBase {
  type: TelemetryEventType;
  appName: string;
  appVersion: string;
  timestamp: string;
  userIdHash?: string | null;
}

export interface UsageEvent extends TelemetryEventBase {
  type: "usage";
  eventName: string;
  properties?: Record<string, unknown>;
}

export interface PerformanceEvent extends TelemetryEventBase {
  type: "performance";
  metricName: string;
  durationMs: number;
  properties?: Record<string, unknown>;
}

export interface ErrorEvent extends TelemetryEventBase {
  type: "error";
  errorName: string;
  message: string;
  stack?: string;
  properties?: Record<string, unknown>;
}

export interface SystemEvent extends TelemetryEventBase {
  type: "system";
  platform: string;
  runtime: string;
  properties?: Record<string, unknown>;
}

export interface NetworkEvent extends TelemetryEventBase {
  type: "network";
  requestId?: string;
  url: string;
  method: string;
  status?: number;
  durationMs?: number;
  properties?: Record<string, unknown>;
}

export interface CustomEvent extends TelemetryEventBase {
  type: "custom";
  eventName: string;
  properties?: Record<string, unknown>;
}

export type TelemetryEvent =
  | UsageEvent
  | PerformanceEvent
  | ErrorEvent
  | SystemEvent
  | NetworkEvent
  | CustomEvent;

export class HiddenConversationTelemetry {
  private config: TelemetryConfig;

  constructor(config: TelemetryConfig) {
    this.config = {
      timeoutMs: 2000,
      ...config,
    };
  }

  private buildBase(type: TelemetryEventType): TelemetryEventBase {
    return {
      type,
      appName: this.config.appName,
      appVersion: this.config.appVersion,
      timestamp: new Date().toISOString(),
      userIdHash: this.config.userIdHash ?? null,
    };
  }

  private async send(event: TelemetryEvent): Promise<void> {
    if (!this.config.telemetryEnabled) return;

    const controller = new AbortController();
    const timeout = setTimeout(
      () => controller.abort(),
      this.config.timeoutMs
    );

    try {
      await fetch(this.config.endpoint, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(event),
        signal: controller.signal,
      });
    } catch {
      // ignore telemetry failures
    } finally {
      clearTimeout(timeout);
    }
  }

  // Usage telemetry
  public trackUsage(eventName: string, properties?: Record<string, unknown>) {
    const event: UsageEvent = {
      ...this.buildBase("usage"),
      eventName,
      properties,
    };
    return this.send(event);
  }

  // Performance telemetry
  public trackPerformance(
    metricName: string,
    durationMs: number,
    properties?: Record<string, unknown>
  ) {
    const event: PerformanceEvent = {
      ...this.buildBase("performance"),
      metricName,
      durationMs,
      properties,
    };
    return this.send(event);
  }

  // Error telemetry
  public trackError(
    error: unknown,
    properties?: Record<string, unknown>
  ) {
    let name = "UnknownError";
    let message = String(error);
    let stack: string | undefined;

    if (error instanceof Error) {
      name = error.name;
      message = error.message;
      stack = error.stack;
    }

    const event: ErrorEvent = {
      ...this.buildBase("error"),
      errorName: name,
      message,
      stack,
      properties,
    };
    return this.send(event);
  }

  // System telemetry
  public trackSystem(properties?: Record<string, unknown>) {
    const platform =
      typeof navigator !== "undefined"
        ? navigator.userAgent
        : typeof process !== "undefined"
        ? `${process.platform} ${process.version}`
        : "unknown";

    const runtime =
      typeof window !== "undefined"
        ? "browser"
        : typeof process !== "undefined"
        ? "node"
        : "unknown";

    const event: SystemEvent = {
      ...this.buildBase("system"),
      platform,
      runtime,
      properties,
    };
    return this.send(event);
  }

  // Network telemetry
  public trackNetwork(
    data: Omit<NetworkEvent, keyof TelemetryEventBase | "type">
  ) {
    const event: NetworkEvent = {
      ...this.buildBase("network"),
      ...data,
    };
    return this.send(event);
  }

  // Custom telemetry
  public trackCustom(
    eventName: string,
    properties?: Record<string, unknown>
  ) {
    const event: CustomEvent = {
      ...this.buildBase("custom"),
      eventName,
      properties,
    };
    return this.send(event);
  }
}

// Example usage:
//
// const telemetry = new HiddenConversationTelemetry({
//   endpoint: "https://your-endpoint.example.com/events",
//   appName: "hidden-conversation",
//   appVersion: "1.0.0",
//   telemetryEnabled: true, // only if user has consented
//   userIdHash: null,
// });
//
// telemetry.trackUsage("app_started", { mode: "cli" });
