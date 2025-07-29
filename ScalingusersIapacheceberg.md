# Stress Testing for the Fediverse with Iceberg 🗻

This is a Rust-based CLI application designed to stress test an [ActivityPub](https://www.w3.org/TR/activitypub/) implementation, simulating user activity (e.g., creating users or posts) while storing metadata in an [Apache Iceberg](https://iceberg.apache.org/) table. The tool is built to handle large-scale testing, with considerations for simulating up to **100 million concurrent users** using asynchronous programming, batch processing, and distributed execution.


# Stress Testing for the Fediverse with Iceberg 🗻

This is a Rust-based CLI application designed to stress test an [ActivityPub](https://www.w3.org/TR/activitypub/) implementation, simulating user activity (e.g., creating users or posts) while storing metadata in an [Apache Iceberg](https://iceberg.apache.org/) table. The tool is built to handle large-scale testing, with considerations for simulating up to **100 million concurrent users** using asynchronous programming, batch processing, and distributed execution.

> **Note**: Simulating 100 million simultaneous users requires a distributed setup (e.g., Kubernetes, AWS ECS). This example starts with a smaller scale (e.g., thousands of users) and provides guidance for scaling up.

## Overview

The stress tester sends HTTP requests to an ActivityPub server (e.g., to create users or posts) and logs user metadata to an Iceberg table for persistent storage and analysis. It uses [Tokio](https://tokio.rs/) for asynchronous execution, [Reqwest](https://crates.io/crates/reqwest) for HTTP requests, and a placeholder for Iceberg integration (to be replaced with an actual Rust Iceberg client if available).

### Assumptions and Constraints
- **ActivityPub Server**: Assumes a running server at `http://localhost:8080` with endpoints like `/users` or `/create_post`.
- **Iceberg Integration**: Uses a placeholder for Iceberg writes, as a Rust client may not exist. Replace with an actual client or JVM-based integration via JNI/REST.
- **Scale**: Simulating 100M users requires distributed infrastructure (e.g., AWS, Kubernetes). This example starts with thousands of users.
- **Authentication**: Assumes simple or no authentication for testing. Adjust for your server’s requirements.
- **Dependencies**: Uses `tokio`, `reqwest`, `clap`, and others for async, HTTP, and CLI functionality.

## Installation

1. **Clone the Repository**:
   ```bash
   git clone https://github.com/your-username/activitypub-stress-tester.git
   cd activitypub-stress-tester
   ```

2. **Add Dependencies**:
   Update your `Cargo.toml` with:
   ```toml
   [dependencies]
   clap = { version = "4.5", features = ["derive"] }
   reqwest = { version = "0.12", features = ["json"] }
   tokio = { version = "1.40", features = ["full"] }
   serde = { version = "1.0", features = ["derive"] }
   chrono = "0.4"
   rand = "0.8"
   futures = "0.3"
   ```

3. **Build the Project**:
   ```bash
   cargo build --release
   ```

## Usage

1. **Start Your ActivityPub Server**:
   Ensure your server is running (e.g., at `http://localhost:8080`) and supports POST requests to endpoints like `/users`.

2. **Configure Iceberg**:
   - If a Rust Iceberg client exists, configure it to connect to your catalog (e.g., AWS Glue, local filesystem).
   - Otherwise, use a REST catalog or JVM-based client via `rust-jni`.

3. **Run the Stress Tester**:
   ```bash
   cargo run --release -- --server http://localhost:8080 --users 1000 --concurrency 100
   ```
   - `--server`: ActivityPub server URL (default: `http://localhost:8080`).
   - `--users`: Number of users to simulate (default: 1000).
   - `--concurrency`: Number of concurrent requests (default: 100).

### Example Output
```bash
$ cargo run --release -- --server http://localhost:8080 --users 1000 --concurrency 100
Created user: user1
Writing to Iceberg: UserRecord { user_id: "1", username: "user1", created_at: "2025-07-28T19:45:00Z" }
...
Created user: user1000
Writing to Iceberg: UserRecord { user_id: "1000", username: "user1000", created_at: "2025-07-28T19:45:05Z" }
Completed 1000 user creations in 5.23 seconds
```

## Code

Below is the main Rust program (`src/main.rs`):

```rust
use clap::{Arg, Command};
use reqwest::Client;
use serde::{Deserialize, Serialize};
use tokio::sync::Semaphore;
use std::sync::Arc;
use std::time::{Duration, Instant};
use rand::Rng;

// Hypothetical Iceberg Rust client (replace with actual crate if available)
mod iceberg {
    use std::error::Error;
    #[derive(Debug, Serialize)]
    pub struct UserRecord {
        pub user_id: String,
        pub username: String,
        pub created_at: String,
    }

    pub async fn write_user_to_iceberg(record: UserRecord) -> Result<(), Box<dyn Error>> {
        // Placeholder: Implement Iceberg write logic here
        println!("Writing to Iceberg: {:?}", record);
        Ok(())
    }
}

#[derive(Debug, Serialize, Deserialize)]
struct ActivityPubUser {
    id: String,
    username: String,
    password: String,
}

async fn create_user(
    client: &Client,
    server_url: &str,
    user_id: u64,
    semaphore: &Semaphore,
) -> Result<(), Box<dyn std::error::Error>> {
    let _permit = semaphore.acquire().await?;
    
    let username = format!("user{}", user_id);
    let user = ActivityPubUser {
        id: user_id.to_string(),
        username: username.clone(),
        password: "test123".to_string(),
    };

    // Send request to ActivityPub server
    let response = client
        .post(&format!("{}/users", server_url))
        .json(&user)
        .send()
        .await?;

    if response.status().is_success() {
        // Write user metadata to Iceberg
        let record = iceberg::UserRecord {
            user_id: user.id,
            username: user.username,
            created_at: chrono::Utc::now().to_rfc3339(),
        };
        iceberg::write_user_to_iceberg(record).await?;
        println!("Created user: {}", username);
    } else {
        eprintln!("Failed to create user {}: {}", username, response.status());
    }

    Ok(())
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // CLI arguments
    let matches = Command::new("activitypub_stress_tester")
        .version("0.1.0")
        .about("Stress tester for ActivityPub implementation with Iceberg storage")
        .arg(
            Arg::new("server")
                .short('s')
                .long("server")
                .default_value("http://localhost:8080")
                .help("ActivityPub server URL"),
        )
        .arg(
            Arg::new("users")
                .short('u')
                .long("users")
                .default_value("1000")
                .help("Number of users to simulate"),
        )
        .arg(
            Arg::new("concurrency")
                .short('c')
                .long("concurrency")
                .default_value("100")
                .help("Number of concurrent requests"),
        )
        .get_matches();

    let server_url = matches.get_one::<String>("server").unwrap();
    let num_users: u64 = matches.get_one::<String>("users").unwrap().parse()?;
    let concurrency: usize = matches.get_one::<String>("concurrency").unwrap().parse()?;

    // Initialize HTTP client
    let client = Client::builder()
        .timeout(Duration::from_secs(10))
        .build()?;

    // Semaphore to limit concurrency
    let semaphore = Arc::new(Semaphore::new(concurrency));

    // Track start time
    let start = Instant::now();

    // Spawn tasks for user creation
    let mut tasks = vec![];
    for i in 1..=num_users {
        let client = client.clone();
        let server_url = server_url.to_string();
        let semaphore = semaphore.clone();
        
        tasks.push(tokio::spawn(async move {
            // Add jitter to avoid thundering herd
            let jitter = rand::thread_rng().gen_range(0..100);
            tokio::time::sleep(Duration::from_millis(jitter)).await;
            
            if let Err(e) = create_user(&client, &server_url, i, &semaphore).await {
                eprintln!("Error creating user {}: {}", i, e);
            }
        }));

        // Process in batches to avoid overwhelming the system
        if tasks.len() >= concurrency {
            futures::future::join_all(tasks).await;
            tasks = vec![];
        }
    }

    // Wait for remaining tasks
    futures::future::join_all(tasks).await;

    let duration = start.elapsed();
    println!(
        "Completed {} user creations in {:.2} seconds",
        num_users,
        duration.as_secs_f64()
    );

    Ok(())
}
```

## How It Works

1. **CLI Arguments**:
   - `--server`: ActivityPub server URL (default: `http://localhost:8080`).
   - `--users`: Number of users to simulate (default: 1000).
   - `--concurrency`: Number of concurrent requests (default: 100).

2. **Concurrency Control**:
   - Uses a `Semaphore` to limit concurrent HTTP requests, preventing resource exhaustion.
   - Processes users in batches to manage memory and CPU usage.

3. **ActivityPub Interaction**:
   - Sends POST requests to the `/users` endpoint to create users.
   - Assumes a JSON payload with `id`, `username`, and `password` fields (adjust for your server’s API).

4. **Iceberg Integration**:
   - Placeholder `iceberg` module for writing user metadata (`user_id`, `username`, `created_at`).
   - Replace with a real Iceberg client or JVM-based wrapper (e.g., via `rust-jni`).

5. **Scalability**:
   - Uses `tokio` for efficient async execution.
   - Adds random jitter to avoid thundering herd issues.
   - Reports total time and errors for monitoring.

## Scaling to 100 Million Users

Simulating 100M simultaneous users is not feasible on a single machine. Here’s how to scale:

- **Distributed Execution**:
  - Deploy on a cluster (e.g., Kubernetes, AWS ECS).
  - Use frameworks like [balter](https://www.reddit.com/r/rust/comments/19fg9dz/balter_01_a_distributed_loadstress_testing/) or `rayon` to distribute tasks.
  - Example: 100 nodes, each handling 1M users.

- **Iceberg Optimization**:
  - Batch writes (e.g., 1000 records per transaction).
  - Use a high-performance catalog (e.g., AWS Glue) and storage (e.g., S3).
  - Partition tables by `created_at` or `user_id`.

- **ActivityPub Server Load**:
  - Ensure horizontal scalability with microservices or serverless architecture.
  - Use a load balancer and monitor with [Prometheus](https://prometheus.io/) or [Grafana](https://grafana.com/).

- **Rate Limiting and Backoff**:
  - Implement exponential backoff for server rate limits.
  - Use [governor](https://crates.io/crates/governor) for rate limiting.

- **Resource Estimation**:
  - Data size: ~10-100 TB for 100M users (depending on record size).
  - Network: 10-100 Gbps bandwidth.
  - Cloud: Use high-memory instances (e.g., AWS EC2, GCP Compute).

## Notes and Limitations

- **Iceberg Placeholder**: Replace the `iceberg` module with a real client or REST/JNI integration.
- **ActivityPub Protocol**: Adjust HTTP requests for your server’s API (e.g., ActivityStreams JSON-LD, authentication).
- **Error Handling**: Basic handling included; add retries and logging for production.
- **Performance Tuning**: Use [drill](https://github.com/fcsonline/drill) or [goose](https://book.goose.rs/) for benchmarking.
- **Realistic Simulation**: Extend to include posting, following, or liking for comprehensive testing.

## References

- [Drill](https://github.com/fcsonline/drill): HTTP load testing tool in Rust.
- [Goose](https://book.goose.rs/): Scalable load testing framework in Rust.
- [Balter](https://www.reddit.com/r/rust/comments/19fg9dz/balter_01_a_distributed_loadstress_testing/): Distributed load testing in Rust.
- [ActivityPub Rust](https://www.reddit.com/r/rust/comments/vnc7f1/presenting_activitypubrust_crate/): Rust library for ActivityPub.

## Contributing

Contributions are welcome! Please submit issues or pull requests to improve the tool, add Iceberg integration, or enhance ActivityPub testing scenarios.

## License

[MIT License](LICENSE)

---

For help with Iceberg integration, distributed scaling, or ActivityPub details, open an issue or contact the maintainers!
``` = "0.3"
