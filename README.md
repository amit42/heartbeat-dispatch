# heartbeat-dispatch

Simple backend where devices check in periodically and receive pending commands or firmware updates in response.

## Overview

This project implements a heartbeat-driven control loop for managing a fleet of devices.

Devices send a periodic heartbeat to the server. Based on the device state and active deployments, the server decides whether to return an action (for example, a firmware update) or no-op.

There are no persistent connections or push mechanisms. Everything is driven by the device polling the backend.

## API documentation

- [Heartbeat API](docs/heartbeat-api.md) — `POST /v1/heartbeat` and request/response details

## Core ideas

- Devices are stateless clients that periodically report their state.
- The server acts as a control plane and decides what each device should do next.
- Deployment decisions are evaluated on every heartbeat.
- Retry and rollout logic is handled entirely on the backend.

## Features

- Heartbeat-based command delivery
- Attribute-based targeting for deployments
- Rollout control using deterministic selection
- Retry with backoff (server-driven)
- Basic deployment health checks (offline detection)
