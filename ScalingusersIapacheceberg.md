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
