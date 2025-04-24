| id   | status    | title                     | description                                 | applies-to            |
|------|-----------|---------------------------|---------------------------------------------|-----------------------|
| OTEL | ACCEPTED  | OpenTelemetry Integration | Specification for OpenTelemetry Integration | client-sdk,server-sdk |

**_See Also:_**

| Link                                       | Description          |
|--------------------------------------------|----------------------|
| [OpenTelemetry Feature Flag in Spans](https://opentelemetry.io/docs/specs/semconv/attributes-registry/feature-flag/) | OpenTelemetry feature flag semantics for spans. |
| [Hooks](../HOOK-hooks/README.md) | Support for hooks in SDKs. |

# 1. OpenTelemetry Integration

## Introduction

It would benefit our customers if they could enhance their observability data with information about flag evaluations. LaunchDarkly could also use SDK telemetry data to build features that correlate flag evaluations with other observed data such as latency and errors.

OpenTelemetry Integration packages will build upon the [Hooks](../HOOK-hooks/README.md) specification to enrich OpenTelemetry data with feature flag evaluations.

OpenTelemetry supports multiple types of telemetry data, including:
- Traces: Provide an overview of how your application handles requests.
  - Feature flag "X" was evaluated for context "Y" while handling this request.
  - Evaluating flag "X" took 100µs.
  - Accessing flags from the redis persistent store added 1ms of latency.
- Metrics: Measure aspects of your application. It may not be correlated with a specific request.
  - X events have been flushed.
  - The SDK encountered X network errors attempting to flush events.
  - The SDK has evaluated X flags.
- Logs: A timestamped textual record containing structured or unstructured data.
  - A flag has been evaluated.
  - The flag configuration has changed.
  - An error has been encountered.

For now our integration packages are going to focus on tracing use cases. The packages will be designed to allow the addition of other use cases.

## 1.1 OpenTelemetry Integration

### Condition 1.1.1

> The implementation technology supports packaging and dependent packages.

### Conditional Requirement 1.1.1.1

> The integration **MUST** be separately installable from the SDK package.

When possible, the OpenTelemetry integration should be a separate package from the SDK package. However, this may not be possible in all implementations (condition 1.1.1).

### Condition 1.1.2

> The implementation technology supports versioned packages and allows for version ranges.

#### Conditional Requirement 1.1.2.1

> The integration **MUST** be forward compatible with new SDK versions and new OpenTelemetry API packages.

The support for the OpenTelemetry API package should be as broad as possible. Requiring an OpenTelemetry API package that is too recent could block adoption.

The implementation should use an approach similar to NPM peer dependencies when possible. Peer dependencies require the application to install the dependency directly, and then the package will use those versions (assuming they meet the semantic versioning requirements).

### Requirement 1.1.2

> The integration **MUST** provide a tracing hook implementation.

## 1.2 Tracing Hook

### Requirement 1.2.1

> The tracing hook **MUST** be implemented as a hook.

Hooks specification: [Hooks](../HOOK-hooks/README.md)

### Requirement 1.2.2

> The tracing hook **MUST** implement support for the `feature_flag` span event.

Reference: [OpenTelemetry Span Events](https://opentelemetry.io/docs/concepts/signals/traces/#span-events).

### Requirement 1.2.2.1

> The `feature_flag` span event **MUST** be added to the active span in the `afterEvaluation` stage.

The `feature_flag` event is generated in the `afterEvaluation` stage so that it may incorporate the result of the evaluation.

Example span event:
```typescript
 {
      name: 'feature_flag',
      attributes: {
        'feature_flag.key': 'my-boolean-flag',
        'feature_flag.provider_name': 'LaunchDarkly Tracing Hook',
        'feature_flag.context.key': 'bob'
      },
      time: [ 1710285750, 507739034 ], // Added by otel API.
      droppedAttributesCount: 0 // Added by otel API.
    }
```

The OpenTelemetry APIs have means to get the active span from the OpenTelemetry context: [Context Interaction](https://opentelemetry.io/docs/specs/otel/trace/api/#context-interaction)

Example:
```typescript
const currentTrace = trace.getActiveSpan();
```

### Requirement 1.2.2.1.1

> If there is no active span, then the hook **MUST** not create the span event.

Example:
```
    const currentTrace = trace.getActiveSpan();
    if (currentTrace) {
      currentTrace.addEvent(FEATURE_FLAG_SCOPE, {
        [FEATURE_FLAG_KEY_ATTR]: flagKey,
        [FEATURE_FLAG_PROVIDER_ATTR]: 'LaunchDarkly',
        [FEATURE_FLAG_CONTEXT_KEY]: canonicalKey,
      });
    }
```

### Requirement 1.2.2.2

> The feature_flag event **MUST** have the following attributes: `feature_flag.key`, `feature_flag.provider_name`, and `feature_flag.context.key`.

### Requirement 1.2.2.3

> The `feature_flag.key` attribute must be set to the key of the flag being evaluated.

This attribute is part of the OpenTelemetry semantic conventions for feature flags.

### Requirement 1.2.2.4

> The `feature_flag.provider_name` attribute must be set to `LaunchDarkly`.

This attribute is part of the OpenTelemetry semantic conventions for feature flags.

### Requirement 1.2.2.5

> The `feature_flag.context.key` attribute must be set to the canonical key of the context the flag is being evaluated for.

This attribute is NOT part of the OpenTelemetry semantic conventions for feature flags.

### Requirement 1.2.2.6

> The `feature_flag` span event **MUST** support an optional `feature_flag.variant`.

This attribute is part of the OpenTelemetry semantic conventions for feature flags.

### Requirement 1.2.2.7

> The `feature_flag.variant` **MUST** be configured at hook registration/construction and default to disabled.

Example:
```
const client = init('sdk-key', { hooks: [new TracingHook({includeVariant: true})] });
```

### Requirement 1.2.2.8

> When enabled the `feature_flag.variant` **MUST** contain the evaluated value of the flag as a string.

Some vendors support semantic names for evaluations for instance, the variant could be `red`, and the flag's value may be `#FF0000`. The OpenTelemetry semantic conventions support a fallback in this case.

```
SHOULD be a semantic identifier for a value. If one is unavailable, a stringified version of the value can be used.
```

### Requirement 1.2.2.9

> The `feature_flag` span event **MUST** support an optional `feature_flag.set.id`.

This attribute is part of the OpenTelemetry semantic conventions for feature flags.

### Condition 1.2.2.9.1

> The `environmentId` is specified in the configuration.

### Conditional Requirement 1.2.2.9.1.1

> The `feature_flag.set.id` **MUST** contain the `environmentId` specified in the configuration.

### Condition 1.2.2.9.2

> The `environmentId` is provided in the `EvaluationSeriesContext`, and is not provided in the configuration.

### Conditional Requirement 1.2.2.9.1.1

> The `feature_flag.set.id` **MUST** contain the `environmentId` provided in the `EvaluationSeriesContext`.

If the `environmentId` is specified in both the configuration, and in the `EvaluationSeriesContext`, then the one provided in the configuration will take precedence.

### Types

### feature_flag span event

- `feature_flag.key` (string, required)
- `feature_flag.context.key` (string, required)
- `feature_flag.provider_name` (string, required)
- `feature_flag.variant` (string, optional)

### Requirement 1.2.3

> The tracing hook **MUST** implement support for optionally creating spans for variation calls.

### Requirement 1.2.3.1

> The creation of spans **MUST** be configured at hook registration/construction and default to disabled.

Example:
```
const client = init('sdk-key', { hooks: [new TracingHook({spans: true})] });
```

### Requirement 1.2.3.2

> The span for the variation method **SHOULD** be active if possible. The span **MUST** be completed before the span event is added.

Becoming the active span can interfere with [1.2.2.1](#requirement-1221); we do not want to attach span events to the span we create. This is why the span must be completed before creating the event, so the event can be attached to the original active span.

### Requirement 1.2.3.3

> The span for the variation method **MUST** support `feature_flag.context.key` and `feature_flag.key` attributes.

The attributes have the same values as those for span events specified in [1.2.2.3](#requirement-1223) and [1.2.2.5](#requirement-1225).

### Requirement 1.2.3.4

> The span **MUST** be created in the `beforeEvaluation` stage of the hook.

The span can be passed to the `afterEvaluation` hook using [EvaluationHookData](../HOOK-hooks/README.md#evaluationhookdata).

### Requirement 1.2.3.5

> The span **MUST** end in the `afterEvaluation` stage of the hook.

### Requirement 1.2.3.6

> The span **MUST** be named after a combination of the client name and the variation method being called.

Examples:
- `LDClient.variationBool`
- `LDClient.variationBoolDetail`
- `LDClient.migrationVariation`

Example span:
```
{
  traceId: 'ce9aab58724853c2c79220f6b4485803', // Added by otel API.
  parentId: '3f239c9e5d9578fd', // Added by otel API.
  traceState: undefined, // Added by otel API.
  name: 'LDClient.stringVariationDetail',
  id: '518bee08360a4021', // Added by otel API.
  kind: 0, // Added by otel API.
  timestamp: 1710285900255000, // Added by otel API.
  duration: 527.398, // Added by otel API.
  attributes: {
    'feature_flag.context.key': 'org:martmart:user:sally',
    'feature_flag.key': 'string-variation'
  },
  status: { code: 0 }, // Added by otel API.
  events: [], // Added by otel API.
  links: [] // Added by otel API.
}
```


### Requirement 1.2.3.7

> The span **MUST** be created against the currently active context.

Using the active context will ensure the span is associated with the active parent span.

Example:
```
const span = this.tracer.startSpan(hookContext.method, undefined, context.active());
```

### Requirement 1.2.4

> The tracing hook **MUST** support configuring an optional `environmentId` which allows the user to specify the environment ID.

```
hooks: [new TracingHook({environmentId: 'the-environment-id'})]
```

### Requirement 1.2.4.1

> The tracing hook **MUST** validate that the `environmentId` is a non-empty string.

If the id is not valid, then it will be equivalent to not having specified the `environmentId`.

### Requirement 1.2.4.2

> The tracing hook **MUST** log when an invalid `environmentId` is provided.
