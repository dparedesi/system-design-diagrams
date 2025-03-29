## Detailed Descriptions

### Real-time audio transcriptions

<details>
  <summary>Click to expand detailed description</summary>

  **Problem:**

  The initial primary goal is to provide real-time transcription for business meetings and conference calls, primarily targeting enterprise users integrated into existing collaboration platforms. Think Zoom, Chime, Teams integrations. When we say "real-time," we mean perceived real-time. The transcribed text for a spoken phrase should appear within 1-2 seconds after the phrase is uttered. It needs to feel interactive. Sub-second would be ideal, but let's target 1-2 seconds as the P99 latency.
  
  For input sources, let's focus initially on live audio streams coming from microphones via web clients or potentially directly from VoIP systems via SIP/RTP streams pushed to our service. Assume standard formats like PCM audio streamed chunk by chunk. Handling pre-recorded files is a secondary goal for now. The output should be the transcribed text, associated with start and end timestamps for each word or logical phrase. Multi-language support is on the roadmap, but let's assume English only for V1 to keep the scope manageable.
  
  **Functional Requiremnets:**

  - Accept persistent WebSocket connections from clients.
  - Receive audio data chunks over established WebSocket connections.
  - Stream received audio chunks to Amazon Transcribe Streaming API.
  - Receive partial and final transcription results from Amazon Transcribe Streaming asynchronously.
  - Send partial and final transcription results back to the originating client via its WebSocket connection in near real-time.
  - Accept and handle specific "end stream" messages from clients to signal intentional completion.
  - Upon intentional completion, store the final, complete transcript persistently (e.g., in S3).
  - Track active WebSocket connections (e.g., storing Connection IDs in DynamoDB).
  - Update session status upon successful completion (e.g., in DynamoDB).
  - Clean up connection-specific resources (e.g., remove Connection ID from DynamoDB) upon client disconnection (both expected and unexpected).

  **Non-Functional Requirements:**

  - **Low Latency:** Transcription results should be returned to the client with minimal delay to maintain a real-time user experience.
  - **High Scalability:** The system must automatically scale to handle a large and fluctuating number of concurrent users and WebSocket connections.
  - **High Availability:** The service should be resilient to failures in individual components and remain operational.
  - **Cost-Effectiveness:** Leverage pay-per-use pricing models to minimize costs, especially during periods of low or no traffic.
  - **Reliability:** Ensure accurate transcription delivery and persistent storage of final transcripts.
  - **Maintainability:** The architecture should be modular and utilize managed services to reduce operational burden.
  - **Security:** Implement appropriate security measures for the WebSocket API endpoint and grant least privilege access via IAM roles.

  ![View Diagram](diagrams/real-time-audio-transcriptions.png)

  **Trade-offs:**

  - **Compute (AWS Lambda vs. Containers - Fargate/ECS/EKS):**
    *   This architecture uses **AWS Lambda** for compute (handling WebSocket events and interacting with Transcribe/DynamoDB/S3).
    *   **Chosen For:** Direct/simple API Gateway integration, automatic fine-grained scaling, pay-per-use cost model, and lowest operational overhead.
    *   **Trade-off Accepted:** Potential for 'cold start' latency on initial invocation or during rapid scaling, mitigated partially by subsequent requests hitting warm instances. Containers (Fargate/ECS) would offer no cold starts (once running) but have higher idle costs, slower scaling initiation, increased operational complexity, and less direct API Gateway WebSocket integration.
  - **Session State/Metadata Database (Amazon DynamoDB vs. Amazon RDS):**
    *   This architecture uses **Amazon DynamoDB** for storing connection IDs and session status.
    *   **Chosen For:** Excellent scalability for high-volume key-value lookups/updates (getting/updating connection/session info), serverless operation (no instances to manage), flexible schema, and pay-per-request pricing aligns well with the event-driven nature.
    *   **Trade-off Accepted:** DynamoDB is less suited for complex relational queries or multi-item transactions compared to RDS. RDS was deemed overkill and operationally heavier for the simple state management required here.

</details>

### URL shortener service

<details>
  <summary>Click to expand detailed description</summary>

  **Problem:**

  We're going to tackle a familiar problem. The task is to design a URL shortener service open to the public.
  
  **Functional Requiremnets:**

  - URL Shortening: Allows users (via API) to submit a long URL and receive a unique, corresponding short URL.
  - URL Redirection: Redirects users accessing a generated short URL (GET /{short_id}) to the original long URL using an HTTP 301/302 status code.
  - Programmatic API Access: Provides a RESTful HTTP API endpoint (POST /urls) for clients to integrate the shortening functionality.
  - Unique ID Generation: Ensures that each generated short URL identifier (short_id) is unique.
  - Idempotent Shortening: Handles duplicate submission requests (e.g., client retries) for the same shortening operation gracefully without creating multiple entries (assuming correct implementation of the idempotency check).

  **Non-Functional Requirements:**

  - **High Availability:** Designed for high availability (e.g., 99.99% target) through multi-region deployment architecture.
  - **High Scalability (Reads):** Read path (redirections) is designed to handle very high traffic volumes.
  - **High Scalability (Writes):** Write path (shortening) is designed to handle high traffic volumes.
  - **Low-Latency Redirections:** Aims for low user-perceived latency for URL redirections (e.g., target p99 < 100ms).
  - **Low-Latency Shortening:** Aims for low latency for the API call to shorten a URL (e.g., target p99 < 500ms).
  - **Eventual Consistency:** Newly shortened URLs will become available for redirection globally after a short propagation delay (typically seconds).
  - **Durability:** Ensures that stored URL mappings are highly durable and protected against data loss.
  - **Security:** Provides protection against common web attacks and unauthorized access patterns.
  - **Operational Efficiency:** Leverages managed services to reduce the operational burden of scaling, patching, and maintenance.

  ![View Diagram](diagrams/URL-shortener.png)

  **Trade-offs:**

  - Database (DynamoDB vs. RDS): Chose DynamoDB for superior key-value scalability & latency over SQL features & complex queries.
  - Read Compute (Fargate vs. Lambda): Chose Fargate for more consistent low latency on high-volume reads, trading off Lambda's simplicity.
  - Write Compute (Lambda vs. Fargate): Chose Lambda for operational simplicity & cost-effectiveness on lower-volume writes, as strict latency is less critical.
  - ID Generation (Decoupled vs. Sync): Chose Decoupled Pool for better write performance & resilience at scale, accepting more architectural components.
  - Caching (ElastiCache vs. None): Added ElastiCache for drastically improved read speed & reduced DB load/cost, accepting cache infrastructure cost.
  - Deployment (Multi-Region vs. Single): Chose Multi-Region for High Availability & global low latency, accepting higher complexity & cost.

</details>