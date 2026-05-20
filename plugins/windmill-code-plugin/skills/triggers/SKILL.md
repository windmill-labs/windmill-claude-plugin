---
name: triggers
description: MUST use when configuring triggers.
---

# Windmill Triggers

Triggers allow external events to invoke your scripts and flows.

## File Naming

Trigger configuration files use the pattern: `{path}.{trigger_type}_trigger.yaml`

Examples:
- `u/user/webhook.http_trigger.yaml`
- `f/data/kafka_consumer.kafka_trigger.yaml`
- `f/sync/postgres_cdc.postgres_trigger.yaml`
- `f/inbound/orders.email_trigger.yaml`

## Email Triggers

An email trigger routes incoming emails to a script or flow. Each trigger reserves a local-part: emails sent to `<local_part>@<windmill_email_domain>` are delivered to the configured runnable. Set `workspaced_local_part: true` to namespace it per workspace (the actual recipient becomes `<workspace_id>-<local_part>@…`); on Windmill Cloud this is required.

Senders may append URL-style extras to the local-part with `+`: `mytrigger+foo=bar+baz=qux@…`. They flow through to the script as `email_extra_args` (see below).

### Payload

The runnable receives:

- `parsed_email` — `{ headers, text_body, html_body, attachments[] }`. Each `attachment` has `{ headers, body }`.
- `raw_email` — the raw RFC 822 message as a string, **or** an S3 object (`{ s3: "windmill_emails/<job_id>/raw.eml" }`) if the message exceeds 1 MiB.
- `email_extra_args` (optional, only when sender appended `+key=value` extras) — a flat object of the parsed extras.

With a preprocessor, all of the above are nested under `event` along with `event.kind = "email"` and `event.trigger_path` (the trigger's path). Without a preprocessor, `trigger_path` is **not** exposed — add a preprocessor if you need it.

### Attachments are S3 objects

Binary attachments are uploaded to the workspace S3 bucket and surface in `parsed_email.attachments[i].body` as:

```json
{ "s3": "windmill_emails/<job_id>/attachments/<filename>" }
```

To read the bytes inside a script, use the wmill SDK:

```ts
// TypeScript
import * as wmill from "windmill-client"
const file = await wmill.loadS3File(parsed_email.attachments[0].body)
```

```python
# Python
import wmill
data = wmill.load_s3_file(parsed_email["attachments"][0]["body"])
```

If the workspace has no S3 resource configured (Workspace Settings → Object storage), `body` falls back to the string `"configure s3 in the workspace settings to handle attachments"`. The same applies to large `raw_email` bodies. Email attachment storage requires the server to be built with the `parquet` feature.

Text/HTML/inline parts are placed inline in `body` as strings.

## CLI Commands

After writing, tell the user they can run these commands (do NOT run them yourself):

```bash
# Push trigger configuration
wmill sync push

# Pull triggers from Windmill
wmill sync pull
```


## HttpTrigger (`*.http_trigger.yaml`)

Must be a YAML file that adheres to the following schema:

```yaml
type: object
properties:
  script_path:
    type: string
    description: Path to the script or flow to execute when triggered
  permissioned_as:
    type: string
    description: The user or group this trigger runs as (permissioned_as)
  is_flow:
    type: boolean
    description: True if script_path points to a flow, false if it points to a script
  labels:
    type: array
    items:
      type: string
  route_path:
    type: string
    description: The URL route path that will trigger this endpoint (e.g., 'api/myendpoint').
      Must NOT start with a /.
  static_asset_config:
    type: object
    properties:
      s3:
        type: string
        description: S3 bucket path for static assets
      storage:
        type: string
        description: Storage path for static assets
      filename:
        type: string
        description: Filename for the static asset
    description: Configuration for serving static assets (s3 bucket, storage path,
      filename)
  http_method:
    type: string
    enum:
    - get
    - post
    - put
    - delete
    - patch
  authentication_resource_path:
    type: string
    description: Path to the resource containing authentication configuration (for
      api_key, basic_http, custom_script, signature methods)
  summary:
    type: string
    description: Short summary describing the purpose of this trigger
  description:
    type: string
    description: Detailed description of what this trigger does
  request_type:
    type: string
    enum:
    - sync
    - async
    - sync_sse
  authentication_method:
    type: string
    enum:
    - none
    - windmill
    - api_key
    - basic_http
    - custom_script
    - signature
  is_static_website:
    type: boolean
    description: If true, serves static files from S3/storage instead of running a
      script
  workspaced_route:
    type: boolean
    description: If true, the route includes the workspace ID in the path
  wrap_body:
    type: boolean
    description: If true, wraps the request body in a 'body' parameter
  raw_string:
    type: boolean
    description: If true, passes the request body as a raw string instead of parsing
      as JSON
  error_handler_path:
    type: string
    description: Path to a script or flow to run when the triggered job fails
  error_handler_args:
    type: object
    description: The arguments to pass to the script or flow
  retry:
    type: object
    properties:
      constant:
        type: object
        description: Retry with constant delay between attempts
        properties:
          attempts:
            type: integer
            description: Number of retry attempts
          seconds:
            type: integer
            description: Seconds to wait between retries
      exponential:
        type: object
        description: Retry with exponential backoff (delay doubles each time)
        properties:
          attempts:
            type: integer
            description: Number of retry attempts
          multiplier:
            type: integer
            description: Multiplier for exponential backoff
          seconds:
            type: integer
            minimum: 1
            description: Initial delay in seconds
          random_factor:
            type: integer
            minimum: 0
            maximum: 100
            description: Random jitter percentage (0-100) to avoid thundering herd
      retry_if:
        $ref: '#/components/schemas/RetryIf'
    description: Retry configuration for failed module executions
required:
- script_path
- permissioned_as
- is_flow
- route_path
- request_type
- authentication_method
- http_method
- is_static_website
- workspaced_route
- wrap_body
- raw_string
```

## WebsocketTrigger (`*.websocket_trigger.yaml`)

Must be a YAML file that adheres to the following schema:

```yaml
type: object
properties:
  script_path:
    type: string
    description: Path to the script or flow to execute when triggered
  permissioned_as:
    type: string
    description: The user or group this trigger runs as (permissioned_as)
  is_flow:
    type: boolean
    description: True if script_path points to a flow, false if it points to a script
  labels:
    type: array
    items:
      type: string
  url:
    type: string
    description: The WebSocket URL to connect to (can be a static URL or computed
      by a runnable)
  filters:
    type: array
    items:
      type: object
      properties:
        key:
          type: string
        value: {}
    description: Array of key-value filters to match incoming messages (only matching
      messages trigger the script)
  filter_logic:
    type: string
    enum:
    - and
    - or
    description: Logic to apply when evaluating filters. 'and' requires all filters
      to match, 'or' requires any filter to match.
  initial_messages:
    type: array
    items:
      type: object
    description: Messages to send immediately after connecting (can be raw strings
      or computed by runnables)
  url_runnable_args:
    type: object
    description: The arguments to pass to the script or flow
  can_return_message:
    type: boolean
    description: If true, the script can return a message to send back through the
      WebSocket
  can_return_error_result:
    type: boolean
    description: If true, error results are sent back through the WebSocket
  heartbeat:
    type: object
    properties:
      interval_secs:
        type: integer
        minimum: 1
        description: Interval in seconds between heartbeat messages
      message:
        type: string
        description: Message to send as heartbeat. Use {{state}} as a placeholder
          for a value extracted from incoming messages (see state_field).
      state_field:
        type: string
        description: Optional. Top-level JSON field to extract from incoming messages.
          The extracted value replaces {{state}} in the heartbeat message.
  error_handler_path:
    type: string
    description: Path to a script or flow to run when the triggered job fails
  error_handler_args:
    type: object
    description: The arguments to pass to the script or flow
  retry:
    type: object
    properties:
      constant:
        type: object
        description: Retry with constant delay between attempts
        properties:
          attempts:
            type: integer
            description: Number of retry attempts
          seconds:
            type: integer
            description: Seconds to wait between retries
      exponential:
        type: object
        description: Retry with exponential backoff (delay doubles each time)
        properties:
          attempts:
            type: integer
            description: Number of retry attempts
          multiplier:
            type: integer
            description: Multiplier for exponential backoff
          seconds:
            type: integer
            minimum: 1
            description: Initial delay in seconds
          random_factor:
            type: integer
            minimum: 0
            maximum: 100
            description: Random jitter percentage (0-100) to avoid thundering herd
      retry_if:
        $ref: '#/components/schemas/RetryIf'
    description: Retry configuration for failed module executions
required:
- script_path
- permissioned_as
- is_flow
- url
- filters
- can_return_message
- can_return_error_result
```

## KafkaTrigger (`*.kafka_trigger.yaml`)

Must be a YAML file that adheres to the following schema:

```yaml
type: object
properties:
  script_path:
    type: string
    description: Path to the script or flow to execute when triggered
  permissioned_as:
    type: string
    description: The user or group this trigger runs as (permissioned_as)
  is_flow:
    type: boolean
    description: True if script_path points to a flow, false if it points to a script
  labels:
    type: array
    items:
      type: string
  kafka_resource_path:
    type: string
    description: Path to the Kafka resource containing connection configuration
  group_id:
    type: string
    description: Kafka consumer group ID for this trigger
  topics:
    type: array
    items:
      type: string
    description: Array of Kafka topic names to subscribe to
  filters:
    type: array
    items:
      type: object
      properties:
        key:
          type: string
        value: {}
  filter_logic:
    type: string
    enum:
    - and
    - or
    description: Logic to apply when evaluating filters. 'and' requires all filters
      to match, 'or' requires any filter to match.
  auto_offset_reset:
    type: string
    enum:
    - latest
    - earliest
    description: Initial offset behavior when consumer group has no committed offset.
      'latest' starts from new messages only, 'earliest' starts from the beginning.
  auto_commit:
    type: boolean
    description: When true (default), offsets are committed automatically after receiving
      each message. When false, you must manually commit offsets using the commit_offsets
      endpoint.
  error_handler_path:
    type: string
    description: Path to a script or flow to run when the triggered job fails
  error_handler_args:
    type: object
    description: The arguments to pass to the script or flow
  retry:
    type: object
    properties:
      constant:
        type: object
        description: Retry with constant delay between attempts
        properties:
          attempts:
            type: integer
            description: Number of retry attempts
          seconds:
            type: integer
            description: Seconds to wait between retries
      exponential:
        type: object
        description: Retry with exponential backoff (delay doubles each time)
        properties:
          attempts:
            type: integer
            description: Number of retry attempts
          multiplier:
            type: integer
            description: Multiplier for exponential backoff
          seconds:
            type: integer
            minimum: 1
            description: Initial delay in seconds
          random_factor:
            type: integer
            minimum: 0
            maximum: 100
            description: Random jitter percentage (0-100) to avoid thundering herd
      retry_if:
        $ref: '#/components/schemas/RetryIf'
    description: Retry configuration for failed module executions
required:
- script_path
- permissioned_as
- is_flow
- kafka_resource_path
- group_id
- topics
- filters
```

## NatsTrigger (`*.nats_trigger.yaml`)

Must be a YAML file that adheres to the following schema:

```yaml
type: object
properties:
  script_path:
    type: string
    description: Path to the script or flow to execute when triggered
  permissioned_as:
    type: string
    description: The user or group this trigger runs as (permissioned_as)
  is_flow:
    type: boolean
    description: True if script_path points to a flow, false if it points to a script
  labels:
    type: array
    items:
      type: string
  nats_resource_path:
    type: string
    description: Path to the NATS resource containing connection configuration
  use_jetstream:
    type: boolean
    description: If true, uses NATS JetStream for durable message delivery
  stream_name:
    type: string
    description: JetStream stream name (required when use_jetstream is true)
  consumer_name:
    type: string
    description: JetStream consumer name (required when use_jetstream is true)
  subjects:
    type: array
    items:
      type: string
    description: Array of NATS subjects to subscribe to
  error_handler_path:
    type: string
    description: Path to a script or flow to run when the triggered job fails
  error_handler_args:
    type: object
    description: The arguments to pass to the script or flow
  retry:
    type: object
    properties:
      constant:
        type: object
        description: Retry with constant delay between attempts
        properties:
          attempts:
            type: integer
            description: Number of retry attempts
          seconds:
            type: integer
            description: Seconds to wait between retries
      exponential:
        type: object
        description: Retry with exponential backoff (delay doubles each time)
        properties:
          attempts:
            type: integer
            description: Number of retry attempts
          multiplier:
            type: integer
            description: Multiplier for exponential backoff
          seconds:
            type: integer
            minimum: 1
            description: Initial delay in seconds
          random_factor:
            type: integer
            minimum: 0
            maximum: 100
            description: Random jitter percentage (0-100) to avoid thundering herd
      retry_if:
        $ref: '#/components/schemas/RetryIf'
    description: Retry configuration for failed module executions
required:
- script_path
- permissioned_as
- is_flow
- nats_resource_path
- use_jetstream
- subjects
```

## PostgresTrigger (`*.postgres_trigger.yaml`)

Must be a YAML file that adheres to the following schema:

```yaml
type: object
properties:
  script_path:
    type: string
    description: Path to the script or flow to execute when triggered
  permissioned_as:
    type: string
    description: The user or group this trigger runs as (permissioned_as)
  is_flow:
    type: boolean
    description: True if script_path points to a flow, false if it points to a script
  labels:
    type: array
    items:
      type: string
  postgres_resource_path:
    type: string
    description: Path to the PostgreSQL resource containing connection configuration
  publication_name:
    type: string
    description: Name of the PostgreSQL publication to subscribe to for change data
      capture
  replication_slot_name:
    type: string
    description: Name of the PostgreSQL logical replication slot to use
  error_handler_path:
    type: string
    description: Path to a script or flow to run when the triggered job fails
  error_handler_args:
    type: object
    description: The arguments to pass to the script or flow
  retry:
    type: object
    properties:
      constant:
        type: object
        description: Retry with constant delay between attempts
        properties:
          attempts:
            type: integer
            description: Number of retry attempts
          seconds:
            type: integer
            description: Seconds to wait between retries
      exponential:
        type: object
        description: Retry with exponential backoff (delay doubles each time)
        properties:
          attempts:
            type: integer
            description: Number of retry attempts
          multiplier:
            type: integer
            description: Multiplier for exponential backoff
          seconds:
            type: integer
            minimum: 1
            description: Initial delay in seconds
          random_factor:
            type: integer
            minimum: 0
            maximum: 100
            description: Random jitter percentage (0-100) to avoid thundering herd
      retry_if:
        $ref: '#/components/schemas/RetryIf'
    description: Retry configuration for failed module executions
required:
- script_path
- permissioned_as
- is_flow
- postgres_resource_path
- replication_slot_name
- publication_name
```

## MqttTrigger (`*.mqtt_trigger.yaml`)

Must be a YAML file that adheres to the following schema:

```yaml
type: object
properties:
  script_path:
    type: string
    description: Path to the script or flow to execute when triggered
  permissioned_as:
    type: string
    description: The user or group this trigger runs as (permissioned_as)
  is_flow:
    type: boolean
    description: True if script_path points to a flow, false if it points to a script
  labels:
    type: array
    items:
      type: string
  mqtt_resource_path:
    type: string
    description: Path to the MQTT resource containing broker connection configuration
  subscribe_topics:
    type: array
    items:
      type: object
    description: Array of MQTT topics to subscribe to, each with topic name and QoS
      level
  v3_config:
    type: object
    properties:
      clean_session:
        type: boolean
  v5_config:
    type: object
    properties:
      clean_start:
        type: boolean
      topic_alias_maximum:
        type: number
      session_expiry_interval:
        type: number
  client_id:
    type: string
    description: MQTT client ID for this connection
  client_version:
    type: string
    enum:
    - v3
    - v5
  error_handler_path:
    type: string
    description: Path to a script or flow to run when the triggered job fails
  error_handler_args:
    type: object
    description: The arguments to pass to the script or flow
  retry:
    type: object
    properties:
      constant:
        type: object
        description: Retry with constant delay between attempts
        properties:
          attempts:
            type: integer
            description: Number of retry attempts
          seconds:
            type: integer
            description: Seconds to wait between retries
      exponential:
        type: object
        description: Retry with exponential backoff (delay doubles each time)
        properties:
          attempts:
            type: integer
            description: Number of retry attempts
          multiplier:
            type: integer
            description: Multiplier for exponential backoff
          seconds:
            type: integer
            minimum: 1
            description: Initial delay in seconds
          random_factor:
            type: integer
            minimum: 0
            maximum: 100
            description: Random jitter percentage (0-100) to avoid thundering herd
      retry_if:
        $ref: '#/components/schemas/RetryIf'
    description: Retry configuration for failed module executions
required:
- script_path
- permissioned_as
- is_flow
- subscribe_topics
- mqtt_resource_path
```

## SqsTrigger (`*.sqs_trigger.yaml`)

Must be a YAML file that adheres to the following schema:

```yaml
type: object
properties:
  script_path:
    type: string
    description: Path to the script or flow to execute when triggered
  permissioned_as:
    type: string
    description: The user or group this trigger runs as (permissioned_as)
  is_flow:
    type: boolean
    description: True if script_path points to a flow, false if it points to a script
  labels:
    type: array
    items:
      type: string
  queue_url:
    type: string
    description: The full URL of the AWS SQS queue to poll for messages
  aws_auth_resource_type:
    type: string
    enum:
    - oidc
    - credentials
  aws_resource_path:
    type: string
    description: Path to the AWS resource containing credentials or OIDC configuration
  message_attributes:
    type: array
    items:
      type: string
    description: Array of SQS message attribute names to include with each message
  error_handler_path:
    type: string
    description: Path to a script or flow to run when the triggered job fails
  error_handler_args:
    type: object
    description: The arguments to pass to the script or flow
  retry:
    type: object
    properties:
      constant:
        type: object
        description: Retry with constant delay between attempts
        properties:
          attempts:
            type: integer
            description: Number of retry attempts
          seconds:
            type: integer
            description: Seconds to wait between retries
      exponential:
        type: object
        description: Retry with exponential backoff (delay doubles each time)
        properties:
          attempts:
            type: integer
            description: Number of retry attempts
          multiplier:
            type: integer
            description: Multiplier for exponential backoff
          seconds:
            type: integer
            minimum: 1
            description: Initial delay in seconds
          random_factor:
            type: integer
            minimum: 0
            maximum: 100
            description: Random jitter percentage (0-100) to avoid thundering herd
      retry_if:
        $ref: '#/components/schemas/RetryIf'
    description: Retry configuration for failed module executions
required:
- script_path
- permissioned_as
- is_flow
- queue_url
- aws_resource_path
- aws_auth_resource_type
```

## GcpTrigger (`*.gcp_trigger.yaml`)

Must be a YAML file that adheres to the following schema:

```yaml
type: object
properties:
  script_path:
    type: string
    description: Path to the script or flow to execute when triggered
  permissioned_as:
    type: string
    description: The user or group this trigger runs as (permissioned_as)
  is_flow:
    type: boolean
    description: True if script_path points to a flow, false if it points to a script
  labels:
    type: array
    items:
      type: string
  gcp_resource_path:
    type: string
    description: Path to the GCP resource containing service account credentials for
      authentication.
  topic_id:
    type: string
    description: Google Cloud Pub/Sub topic ID to subscribe to.
  subscription_id:
    type: string
    description: Google Cloud Pub/Sub subscription ID.
  delivery_type:
    type: string
    enum:
    - push
    - pull
    description: Delivery mode for messages. 'push' for HTTP push delivery where messages
      are sent to a webhook endpoint, 'pull' for polling where the trigger actively
      fetches messages.
  delivery_config:
    type: object
    properties:
      audience:
        type: string
        description: The audience claim for OIDC tokens used in push authentication.
      authenticate:
        type: boolean
        description: If true, push messages will include OIDC authentication tokens.
    description: Configuration for push delivery mode.
  subscription_mode:
    type: string
    enum:
    - existing
    - create_update
    description: The mode of subscription. 'existing' means using an existing GCP
      subscription, while 'create_update' involves creating or updating a new subscription.
  error_handler_path:
    type: string
    description: Path to a script or flow to run when the triggered job fails.
  error_handler_args:
    type: object
    description: The arguments to pass to the script or flow
  retry:
    type: object
    properties:
      constant:
        type: object
        description: Retry with constant delay between attempts
        properties:
          attempts:
            type: integer
            description: Number of retry attempts
          seconds:
            type: integer
            description: Seconds to wait between retries
      exponential:
        type: object
        description: Retry with exponential backoff (delay doubles each time)
        properties:
          attempts:
            type: integer
            description: Number of retry attempts
          multiplier:
            type: integer
            description: Multiplier for exponential backoff
          seconds:
            type: integer
            minimum: 1
            description: Initial delay in seconds
          random_factor:
            type: integer
            minimum: 0
            maximum: 100
            description: Random jitter percentage (0-100) to avoid thundering herd
      retry_if:
        $ref: '#/components/schemas/RetryIf'
    description: Retry configuration for failed module executions
required:
- script_path
- permissioned_as
- is_flow
- gcp_resource_path
- topic_id
- subscription_id
- delivery_type
- subscription_mode
```

## AzureTrigger (`*.azure_trigger.yaml`)

Must be a YAML file that adheres to the following schema:

```yaml
type: object
properties:
  script_path:
    type: string
    description: Path to the script or flow to execute when triggered
  permissioned_as:
    type: string
    description: The user or group this trigger runs as (permissioned_as)
  is_flow:
    type: boolean
    description: True if script_path points to a flow, false if it points to a script
  labels:
    type: array
    items:
      type: string
  azure_resource_path:
    type: string
  azure_mode:
    type: string
    enum:
    - basic_push
    - namespace_push
    - namespace_pull
    description: Azure Event Grid trigger mode.
  scope_resource_id:
    type: string
    description: ARM resource ID of the topic (basic) or namespace (namespace modes).
  topic_name:
    type: string
    description: Topic name within the namespace (namespace modes only).
  subscription_name:
    type: string
  event_type_filters:
    type: array
    items:
      type: string
  error_handler_path:
    type: string
  error_handler_args:
    type: object
    description: The arguments to pass to the script or flow
  retry:
    type: object
    properties:
      constant:
        type: object
        description: Retry with constant delay between attempts
        properties:
          attempts:
            type: integer
            description: Number of retry attempts
          seconds:
            type: integer
            description: Seconds to wait between retries
      exponential:
        type: object
        description: Retry with exponential backoff (delay doubles each time)
        properties:
          attempts:
            type: integer
            description: Number of retry attempts
          multiplier:
            type: integer
            description: Multiplier for exponential backoff
          seconds:
            type: integer
            minimum: 1
            description: Initial delay in seconds
          random_factor:
            type: integer
            minimum: 0
            maximum: 100
            description: Random jitter percentage (0-100) to avoid thundering herd
      retry_if:
        $ref: '#/components/schemas/RetryIf'
    description: Retry configuration for failed module executions
required:
- script_path
- permissioned_as
- is_flow
- azure_resource_path
- azure_mode
- scope_resource_id
- subscription_name
```

## EmailTrigger (`*.email_trigger.yaml`)

Must be a YAML file that adheres to the following schema:

```yaml
type: object
properties:
  script_path:
    type: string
    description: Path to the script or flow to execute when triggered
  permissioned_as:
    type: string
    description: The user or group this trigger runs as (permissioned_as)
  is_flow:
    type: boolean
    description: True if script_path points to a flow, false if it points to a script
  labels:
    type: array
    items:
      type: string
  local_part:
    type: string
  workspaced_local_part:
    type: boolean
  error_handler_path:
    type: string
  error_handler_args:
    type: object
    description: The arguments to pass to the script or flow
  retry:
    type: object
    properties:
      constant:
        type: object
        description: Retry with constant delay between attempts
        properties:
          attempts:
            type: integer
            description: Number of retry attempts
          seconds:
            type: integer
            description: Seconds to wait between retries
      exponential:
        type: object
        description: Retry with exponential backoff (delay doubles each time)
        properties:
          attempts:
            type: integer
            description: Number of retry attempts
          multiplier:
            type: integer
            description: Multiplier for exponential backoff
          seconds:
            type: integer
            minimum: 1
            description: Initial delay in seconds
          random_factor:
            type: integer
            minimum: 0
            maximum: 100
            description: Random jitter percentage (0-100) to avoid thundering herd
      retry_if:
        $ref: '#/components/schemas/RetryIf'
    description: Retry configuration for failed module executions
required:
- script_path
- permissioned_as
- is_flow
- local_part
```