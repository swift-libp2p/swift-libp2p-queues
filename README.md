# Queues

[![](https://img.shields.io/badge/made%20by-Breth-blue.svg?style=flat-square)](https://breth.app)
[![](https://img.shields.io/badge/project-libp2p-yellow.svg?style=flat-square)](http://libp2p.io/)
[![Swift Package Manager compatible](https://img.shields.io/badge/SPM-compatible-blue.svg?style=flat-square)](https://github.com/apple/swift-package-manager)
![Build & Test (macos and linux)](https://github.com/swift-libp2p/swift-libp2p-queues/actions/workflows/build+test.yml/badge.svg)

> A Vapor-esque Queues implementation for your swift-libp2p app

## Table of Contents

- [Overview](#overview)
- [Install](#install)
- [Usage](#usage)
  - [Example](#example)
- [Contributing](#contributing)
- [Credits](#credits)
- [License](#license)

## Overview
This repo contains the code necessary for a swift-libp2p app to create, schedule, dispatch and execute jobs / services. 
- It is designed to be used with a Queues driver (ex: redis queues driver)
- See the Vapor Queues documentation for further information and examples on how to use Queues in your app.

## Install

Include the following dependency in your Package.swift file
```Swift
let package = Package(
    ...
    dependencies: [
        ...
        .package(url: "https://github.com/swift-libp2p/swift-libp2p-queues.git", .upToNextMinor(from: "0.1.0"))
    ],
    ...
        .target(
            ...
            dependencies: [
                ...
                .product(name: "Queues", package: "swift-libp2p-queues"),
            ]),
    ...
)
```

## Usage

### Example 
Check out the [tests](Tests/QueuesTests) for more examples.

#### Defining a Job
```Swift
import Queues

struct EmailJob: AsyncJob {
    // Any `Codable` payload is automatically serialized to / from JSON
    struct Payload: Codable {
        let to: String
        let message: String
    }

    func dequeue(_ context: QueueContext, _ payload: Payload) async throws {
        context.logger.info("Sending email to \(payload.to)")
        // ... do the work
    }

    // Optional: called when `dequeue` throws
    func error(_ context: QueueContext, _ error: any Error, _ payload: Payload) async throws {
        context.logger.error("Failed to send email to \(payload.to): \(error)")
    }

    // Optional: seconds to wait before retrying (defaults to 0)
    func nextRetryIn(attempt: Int) -> Int { attempt * 5 }
}
```

#### Configuring Queues
```Swift
import LibP2P
import Queues

func configure(_ app: Application) throws {
    // Choose a driver (ex: a redis queues driver, or `.test` / `.asyncTest` from `QueuesTesting`)
    app.queues.use(.myDriver)

    // Optional configuration
    app.queues.configuration.refreshInterval = .seconds(2)
    app.queues.configuration.workerCount = 4

    // Register every job type that can be dequeued
    app.queues.add(EmailJob())

    // Run the workers in the same process as your app
    // (alternatively run `swift run App queues` / `swift run App queues --scheduled`)
    try app.queues.startInProcessJobs(on: .default)
}
```

#### Dispatching a Job from a Route
```Swift
app.on("email") { req -> Response<String> in
    switch req.event {
    case .ready:
        return .stayOpen
    case .data:
        try await req.queue.dispatch(
            EmailJob.self,
            .init(to: "peer@example.com", message: "Hello!"),
            maxRetryCount: 3,
            delayUntil: Date().addingTimeInterval(60)
        )
        return .respondThenClose("queued")
    case .closed, .error:
        return .close
    }
}

// Dispatch onto a specific, named queue
extension QueueName {
    static let emails = QueueName(string: "emails")
}

try await req.queues(.emails).dispatch(EmailJob.self, .init(to: "peer@example.com", message: "Hi"))
```

#### Scheduled Jobs
```Swift
struct CleanupJob: AsyncScheduledJob {
    func run(context: QueueContext) async throws {
        context.logger.info("Cleaning up...")
    }
}

app.queues.schedule(CleanupJob()).daily().at(.midnight)
app.queues.schedule(CleanupJob()).weekly().on(.monday).at("3:30pm")
app.queues.schedule(CleanupJob()).yearly().in(.may).on(23).at(.noon)
app.queues.schedule(CleanupJob()).hourly().at(15)
app.queues.schedule(CleanupJob()).everySecond()

try app.queues.startScheduledJobs()
```

#### Job Event Delegates
```Swift
struct JobLogger: AsyncJobEventDelegate {
    let logger: Logger

    func dispatched(job: JobEventData) async throws {
        logger.info("Dispatched \(job.jobName) [\(job.id)] on \(job.queueName)")
    }

    func success(jobId: String) async throws {
        logger.info("Job \(jobId) succeeded")
    }

    func error(jobId: String, error: any Error) async throws {
        logger.warning("Job \(jobId) failed: \(error)")
    }
}

app.queues.add(JobLogger(logger: app.logger))
```

#### Testing
```Swift
import LibP2PTesting
import Queues
import QueuesTesting

app.queues.use(.test)
app.queues.add(EmailJob())

// ... dispatch some jobs

#expect(app.queues.test.queue.count == 1)
#expect(app.queues.test.contains(EmailJob.self))
let payload = try #require(app.queues.test.first(EmailJob.self))

// Drain the queue
try await app.queues.queue.worker.run()
#expect(app.queues.test.queue.isEmpty)
```

## Contributing

Contributions are welcomed! This code is very much a proof of concept. I can guarantee you there's a better / safer way to accomplish the same results. Any suggestions, improvements, or even just critiques, are welcome! 

Let's make this code better together! 🤝

## Credits

- [Vapor Queues](https://github.com/vapor/queues)

## License

[MIT](LICENSE) © 2026 Breth Inc.

