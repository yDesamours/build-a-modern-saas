# Module 2.5: Core Design Patterns for SaaS Applications

---

## Part 4: Behavioral Patterns - Command Pattern

---

## 🎯 Learning Objectives

By the end of this section, you will:

- Understand what the Command pattern is and when to use it
- Recognize scenarios where Command shines in SaaS applications
- Implement Command pattern in Java, Node.js, and Go
- Build undo/redo functionality with command history
- Create job queues and background task systems
- Implement audit logging and transaction management
- Avoid common pitfalls like command coupling and memory issues

---

## 📖 Conceptual Overview

### What is the Command Pattern?

The **Command pattern** encapsulates a request as an object, thereby letting you parameterize clients with different requests, queue or log requests, and support undoable operations.

**Key Idea**: Instead of calling methods directly, wrap the request in an object that can be stored, queued, logged, or undone.

### Real-World Analogy

Think of a restaurant:

- **Customer** (Client): Orders food
- **Waiter** (Invoker): Takes order and gives to kitchen
- **Order Ticket** (Command): Written order with all details
- **Chef** (Receiver): Executes the order

The order ticket can be:

- **Queued**: Multiple orders in queue
- **Logged**: Kitchen keeps records
- **Modified**: Cancel or change order
- **Replayed**: Make same order again

---

## 🤔 When to Use Command Pattern in SaaS

### Perfect Use Cases:

1. **Job Queues & Background Tasks** ⭐⭐⭐

   - Email sending jobs
   - Report generation
   - Data imports/exports
   - Image/video processing
   - Webhook delivery

2. **Undo/Redo Functionality** ⭐⭐⭐

   - Document editors
   - Design tools
   - Configuration management
   - Data entry forms

3. **Audit Logging** ⭐⭐⭐

   - Record every action as command
   - Replay commands for debugging
   - Compliance requirements
   - Change tracking

4. **Transaction Management** ⭐⭐

   - Multi-step operations
   - Distributed transactions
   - Saga pattern implementation
   - Rollback on failure

5. **Scheduled Tasks** ⭐⭐

   - Cron-like jobs
   - Deferred execution
   - Rate-limited operations
   - Batch processing

6. **Macro Recording** ⭐⭐

   - Record sequence of commands
   - Replay automation
   - Batch operations
   - API replay testing

7. **Request Queuing** ⭐⭐⭐

   - Rate limiting
   - Load leveling
   - Priority queues
   - Circuit breaker implementation

8. **Multi-level Undo** ⭐⭐
   - Complex undo stacks
   - Selective undo
   - Undo groups/transactions

---

## 🚫 When NOT to Use Command

- **Simple CRUD operations** - Direct calls are simpler
- **No need for queuing/undo** - Adds unnecessary complexity
- **Stateless operations** - No benefit from encapsulation
- **Real-time direct actions** - Command overhead not needed

---

## 🏗️ Structure

```
Client ──creates──> Command (Interface)
                        ↑
                        |
                   ConcreteCommand ──uses──> Receiver
                        ↑
                        |
Invoker ──stores/executes──> Command
```

**Key Elements:**

1. **Command Interface**: Defines execute() method
2. **ConcreteCommand**: Implements execute(), holds receiver reference
3. **Receiver**: Actual object that performs the work
4. **Invoker**: Stores and executes commands
5. **Client**: Creates command and sets receiver

---

## 💻 Implementation in Java/Spring Boot

### Scenario 1: Job Queue System

Build a background job processing system with retry and failure handling.

#### Step 1: Define Command Interface

```java
// Command.java - Base command interface
package com.saas.command;

public interface Command {
    /**
     * Execute the command
     * @return true if successful, false if failed
     */
    boolean execute();

    /**
     * Undo the command (if supported)
     */
    void undo();

    /**
     * Get command name for logging
     */
    String getName();

    /**
     * Get command metadata
     */
    CommandMetadata getMetadata();
}
```

```java
// CommandMetadata.java
package com.saas.command;

import lombok.Builder;
import lombok.Getter;
import java.time.LocalDateTime;

@Getter
@Builder
public class CommandMetadata {
    private String commandId;
    private String commandType;
    private LocalDateTime createdAt;
    private int retryCount;
    private int maxRetries;
    private String userId;
    private String tenantId;
}
```

#### Step 2: Implement Concrete Commands

```java
// SendEmailCommand.java
package com.saas.command.impl;

import com.saas.command.Command;
import com.saas.command.CommandMetadata;
import com.saas.service.EmailService;
import lombok.extern.slf4j.Slf4j;

@Slf4j
public class SendEmailCommand implements Command {

    private final EmailService emailService;
    private final String to;
    private final String subject;
    private final String body;
    private final CommandMetadata metadata;

    public SendEmailCommand(
            EmailService emailService,
            String to,
            String subject,
            String body,
            CommandMetadata metadata) {
        this.emailService = emailService;
        this.to = to;
        this.subject = subject;
        this.body = body;
        this.metadata = metadata;
    }

    @Override
    public boolean execute() {
        log.info("Executing SendEmailCommand: {} to {}",
            metadata.getCommandId(), to);

        try {
            emailService.sendEmail(to, subject, body);
            log.info("Email sent successfully: {}", metadata.getCommandId());
            return true;
        } catch (Exception e) {
            log.error("Failed to send email: {}", metadata.getCommandId(), e);
            return false;
        }
    }

    @Override
    public void undo() {
        // Cannot undo email sending
        log.warn("Cannot undo email sending: {}", metadata.getCommandId());
    }

    @Override
    public String getName() {
        return "SendEmail";
    }

    @Override
    public CommandMetadata getMetadata() {
        return metadata;
    }
}
```

```java
// GenerateReportCommand.java
package com.saas.command.impl;

import com.saas.command.Command;
import com.saas.command.CommandMetadata;
import com.saas.service.ReportService;
import lombok.extern.slf4j.Slf4j;

@Slf4j
public class GenerateReportCommand implements Command {

    private final ReportService reportService;
    private final String reportType;
    private final String userId;
    private final CommandMetadata metadata;
    private String generatedReportPath;

    public GenerateReportCommand(
            ReportService reportService,
            String reportType,
            String userId,
            CommandMetadata metadata) {
        this.reportService = reportService;
        this.reportType = reportType;
        this.userId = userId;
        this.metadata = metadata;
    }

    @Override
    public boolean execute() {
        log.info("Executing GenerateReportCommand: {} type: {}",
            metadata.getCommandId(), reportType);

        try {
            generatedReportPath = reportService.generateReport(reportType, userId);
            log.info("Report generated: {} at {}",
                metadata.getCommandId(), generatedReportPath);
            return true;
        } catch (Exception e) {
            log.error("Failed to generate report: {}", metadata.getCommandId(), e);
            return false;
        }
    }

    @Override
    public void undo() {
        if (generatedReportPath != null) {
            log.info("Deleting generated report: {}", generatedReportPath);
            reportService.deleteReport(generatedReportPath);
        }
    }

    @Override
    public String getName() {
        return "GenerateReport";
    }

    @Override
    public CommandMetadata getMetadata() {
        return metadata;
    }
}
```

```java
// ImportDataCommand.java
package com.saas.command.impl;

import com.saas.command.Command;
import com.saas.command.CommandMetadata;
import com.saas.service.ImportService;
import lombok.extern.slf4j.Slf4j;
import java.util.List;

@Slf4j
public class ImportDataCommand implements Command {

    private final ImportService importService;
    private final String filePath;
    private final String importType;
    private final CommandMetadata metadata;
    private List<String> importedRecordIds;

    public ImportDataCommand(
            ImportService importService,
            String filePath,
            String importType,
            CommandMetadata metadata) {
        this.importService = importService;
        this.filePath = filePath;
        this.importType = importType;
        this.metadata = metadata;
    }

    @Override
    public boolean execute() {
        log.info("Executing ImportDataCommand: {} file: {}",
            metadata.getCommandId(), filePath);

        try {
            importedRecordIds = importService.importData(filePath, importType);
            log.info("Data imported: {} records: {}",
                metadata.getCommandId(), importedRecordIds.size());
            return true;
        } catch (Exception e) {
            log.error("Failed to import data: {}", metadata.getCommandId(), e);
            return false;
        }
    }

    @Override
    public void undo() {
        if (importedRecordIds != null && !importedRecordIds.isEmpty()) {
            log.info("Undoing import, deleting {} records", importedRecordIds.size());
            importService.deleteRecords(importedRecordIds);
        }
    }

    @Override
    public String getName() {
        return "ImportData";
    }

    @Override
    public CommandMetadata getMetadata() {
        return metadata;
    }
}
```

#### Step 3: Command Queue (Invoker)

```java
// CommandQueue.java
package com.saas.command.queue;

import com.saas.command.Command;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
import java.util.concurrent.*;
import javax.annotation.PreDestroy;

@Component
@Slf4j
public class CommandQueue {

    private final BlockingQueue<Command> queue = new LinkedBlockingQueue<>();
    private final ExecutorService executor = Executors.newFixedThreadPool(5);
    private final ScheduledExecutorService retryExecutor =
        Executors.newSingleThreadScheduledExecutor();
    private final Map<String, Command> failedCommands = new ConcurrentHashMap<>();
    private volatile boolean running = true;

    public CommandQueue() {
        // Start queue processor
        startQueueProcessor();

        // Start retry scheduler
        startRetryScheduler();
    }

    /**
     * Add command to queue
     */
    public void enqueue(Command command) {
        log.info("Enqueuing command: {} ({})",
            command.getName(), command.getMetadata().getCommandId());

        try {
            queue.put(command);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            log.error("Failed to enqueue command", e);
        }
    }

    /**
     * Process commands from queue
     */
    private void startQueueProcessor() {
        executor.submit(() -> {
            while (running) {
                try {
                    Command command = queue.take();
                    processCommand(command);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    log.info("Queue processor interrupted");
                    break;
                }
            }
        });
    }

    /**
     * Process a single command
     */
    private void processCommand(Command command) {
        log.info("Processing command: {} ({})",
            command.getName(), command.getMetadata().getCommandId());

        try {
            boolean success = command.execute();

            if (success) {
                log.info("Command executed successfully: {}",
                    command.getMetadata().getCommandId());

                // Remove from failed commands if it was there
                failedCommands.remove(command.getMetadata().getCommandId());
            } else {
                handleFailedCommand(command);
            }
        } catch (Exception e) {
            log.error("Command execution threw exception: {}",
                command.getMetadata().getCommandId(), e);
            handleFailedCommand(command);
        }
    }

    /**
     * Handle failed command with retry logic
     */
    private void handleFailedCommand(Command command) {
        CommandMetadata metadata = command.getMetadata();

        if (metadata.getRetryCount() < metadata.getMaxRetries()) {
            log.warn("Command failed, will retry: {} (attempt {}/{})",
                metadata.getCommandId(),
                metadata.getRetryCount() + 1,
                metadata.getMaxRetries());

            // Store for retry
            failedCommands.put(metadata.getCommandId(), command);
        } else {
            log.error("Command failed permanently after {} retries: {}",
                metadata.getMaxRetries(),
                metadata.getCommandId());

            // Move to dead letter queue
            moveToDeadLetterQueue(command);
        }
    }

    /**
     * Retry failed commands periodically
     */
    private void startRetryScheduler() {
        retryExecutor.scheduleWithFixedDelay(() -> {
            if (!failedCommands.isEmpty()) {
                log.info("Retrying {} failed commands", failedCommands.size());

                failedCommands.values().forEach(command -> {
                    // Increment retry count
                    CommandMetadata metadata = command.getMetadata();
                    CommandMetadata updatedMetadata = CommandMetadata.builder()
                        .commandId(metadata.getCommandId())
                        .commandType(metadata.getCommandType())
                        .createdAt(metadata.getCreatedAt())
                        .retryCount(metadata.getRetryCount() + 1)
                        .maxRetries(metadata.getMaxRetries())
                        .userId(metadata.getUserId())
                        .tenantId(metadata.getTenantId())
                        .build();

                    // Re-enqueue
                    enqueue(command);
                });

                failedCommands.clear();
            }
        }, 60, 60, TimeUnit.SECONDS); // Retry every 60 seconds
    }

    /**
     * Move permanently failed commands to dead letter queue
     */
    private void moveToDeadLetterQueue(Command command) {
        log.error("Moving command to dead letter queue: {}",
            command.getMetadata().getCommandId());

        // Store in database for manual inspection/retry
        // Implementation depends on your persistence layer
    }

    /**
     * Get queue statistics
     */
    public QueueStats getStats() {
        return QueueStats.builder()
            .queueSize(queue.size())
            .failedCommandsCount(failedCommands.size())
            .build();
    }

    @PreDestroy
    public void shutdown() {
        log.info("Shutting down command queue");
        running = false;
        executor.shutdown();
        retryExecutor.shutdown();

        try {
            if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
                executor.shutdownNow();
            }
            if (!retryExecutor.awaitTermination(10, TimeUnit.SECONDS)) {
                retryExecutor.shutdownNow();
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
            retryExecutor.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }
}
```

```java
// QueueStats.java
package com.saas.command.queue;

import lombok.Builder;
import lombok.Getter;

@Getter
@Builder
public class QueueStats {
    private int queueSize;
    private int failedCommandsCount;
}
```

#### Step 4: Command Service

```java
// CommandService.java
package com.saas.service;

import com.saas.command.*;
import com.saas.command.impl.*;
import com.saas.command.queue.CommandQueue;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import java.time.LocalDateTime;
import java.util.UUID;

@Service
@RequiredArgsConstructor
@Slf4j
public class CommandService {

    private final CommandQueue commandQueue;
    private final EmailService emailService;
    private final ReportService reportService;
    private final ImportService importService;

    /**
     * Schedule email sending
     */
    public String scheduleEmail(String to, String subject, String body, String userId) {
        log.info("Scheduling email to: {}", to);

        CommandMetadata metadata = createMetadata("SendEmail", userId);

        Command command = new SendEmailCommand(
            emailService,
            to,
            subject,
            body,
            metadata
        );

        commandQueue.enqueue(command);

        return metadata.getCommandId();
    }

    /**
     * Schedule report generation
     */
    public String scheduleReportGeneration(String reportType, String userId) {
        log.info("Scheduling report generation: {}", reportType);

        CommandMetadata metadata = createMetadata("GenerateReport", userId);

        Command command = new GenerateReportCommand(
            reportService,
            reportType,
            userId,
            metadata
        );

        commandQueue.enqueue(command);

        return metadata.getCommandId();
    }

    /**
     * Schedule data import
     */
    public String scheduleDataImport(String filePath, String importType, String userId) {
        log.info("Scheduling data import from: {}", filePath);

        CommandMetadata metadata = createMetadata("ImportData", userId);

        Command command = new ImportDataCommand(
            importService,
            filePath,
            importType,
            metadata
        );

        commandQueue.enqueue(command);

        return metadata.getCommandId();
    }

    /**
     * Create command metadata
     */
    private CommandMetadata createMetadata(String commandType, String userId) {
        return CommandMetadata.builder()
            .commandId(UUID.randomUUID().toString())
            .commandType(commandType)
            .createdAt(LocalDateTime.now())
            .retryCount(0)
            .maxRetries(3)
            .userId(userId)
            .build();
    }
}
```

#### Step 5: REST Controller

```java
// JobController.java
package com.saas.controller;

import com.saas.command.queue.QueueStats;
import com.saas.service.CommandService;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/jobs")
@RequiredArgsConstructor
public class JobController {

    private final CommandService commandService;
    private final CommandQueue commandQueue;

    @PostMapping("/email")
    public ResponseEntity<JobResponse> scheduleEmail(@RequestBody EmailRequest request) {
        String jobId = commandService.scheduleEmail(
            request.getTo(),
            request.getSubject(),
            request.getBody(),
            request.getUserId()
        );

        return ResponseEntity.ok(new JobResponse(jobId, "Email job scheduled"));
    }

    @PostMapping("/report")
    public ResponseEntity<JobResponse> scheduleReport(@RequestBody ReportRequest request) {
        String jobId = commandService.scheduleReportGeneration(
            request.getReportType(),
            request.getUserId()
        );

        return ResponseEntity.ok(new JobResponse(jobId, "Report job scheduled"));
    }

    @PostMapping("/import")
    public ResponseEntity<JobResponse> scheduleImport(@RequestBody ImportRequest request) {
        String jobId = commandService.scheduleDataImport(
            request.getFilePath(),
            request.getImportType(),
            request.getUserId()
        );

        return ResponseEntity.ok(new JobResponse(jobId, "Import job scheduled"));
    }

    @GetMapping("/stats")
    public ResponseEntity<QueueStats> getStats() {
        return ResponseEntity.ok(commandQueue.getStats());
    }
}
```

### Scenario 2: Undo/Redo System

```java
// UndoableCommand.java - Command that supports undo
package com.saas.command;

public interface UndoableCommand extends Command {
    /**
     * Check if command can be undone
     */
    boolean canUndo();

    /**
     * Get description of what will be undone
     */
    String getUndoDescription();
}
```

```java
// UpdateUserCommand.java - Undoable command example
package com.saas.command.impl;

import com.saas.command.CommandMetadata;
import com.saas.command.UndoableCommand;
import com.saas.model.User;
import com.saas.repository.UserRepository;
import lombok.extern.slf4j.Slf4j;

@Slf4j
public class UpdateUserCommand implements UndoableCommand {

    private final UserRepository userRepository;
    private final String userId;
    private final String newName;
    private final String newEmail;
    private final CommandMetadata metadata;

    // Store old values for undo
    private String oldName;
    private String oldEmail;
    private boolean executed = false;

    public UpdateUserCommand(
            UserRepository userRepository,
            String userId,
            String newName,
            String newEmail,
            CommandMetadata metadata) {
        this.userRepository = userRepository;
        this.userId = userId;
        this.newName = newName;
        this.newEmail = newEmail;
        this.metadata = metadata;
    }

    @Override
    public boolean execute() {
        log.info("Executing UpdateUserCommand: {}", userId);

        try {
            User user = userRepository.findById(userId)
                .orElseThrow(() -> new RuntimeException("User not found"));

            // Store old values
            this.oldName = user.getName();
            this.oldEmail = user.getEmail();

            // Update user
            user.setName(newName);
            user.setEmail(newEmail);
            userRepository.save(user);

            executed = true;
            log.info("User updated: {}", userId);
            return true;
        } catch (Exception e) {
            log.error("Failed to update user: {}", userId, e);
            return false;
        }
    }

    @Override
    public void undo() {
        if (!executed) {
            log.warn("Cannot undo: command not executed");
            return;
        }

        log.info("Undoing UpdateUserCommand: {}", userId);

        try {
            User user = userRepository.findById(userId)
                .orElseThrow(() -> new RuntimeException("User not found"));

            // Restore old values
            user.setName(oldName);
            user.setEmail(oldEmail);
            userRepository.save(user);

            log.info("User update undone: {}", userId);
        } catch (Exception e) {
            log.error("Failed to undo user update: {}", userId, e);
        }
    }

    @Override
    public boolean canUndo() {
        return executed;
    }

    @Override
    public String getUndoDescription() {
        return String.format("Undo user update: %s -> %s", newName, oldName);
    }

    @Override
    public String getName() {
        return "UpdateUser";
    }

    @Override
    public CommandMetadata getMetadata() {
        return metadata;
    }
}
```

```java
// CommandHistory.java - Undo/Redo manager
package com.saas.command;

import lombok.extern.slf4j.Slf4j;
import java.util.Stack;

@Slf4j
public class CommandHistory {

    private final Stack<UndoableCommand> undoStack = new Stack<>();
    private final Stack<UndoableCommand> redoStack = new Stack<>();
    private final int maxHistorySize;

    public CommandHistory(int maxHistorySize) {
        this.maxHistorySize = maxHistorySize;
    }

    /**
     * Execute command and add to history
     */
    public boolean executeCommand(UndoableCommand command) {
        log.info("Executing command: {}", command.getName());

        boolean success = command.execute();

        if (success && command.canUndo()) {
            undoStack.push(command);
            redoStack.clear(); // Clear redo stack on new command

            // Limit history size
            if (undoStack.size() > maxHistorySize) {
                undoStack.remove(0);
            }

            log.info("Command added to history (size: {})", undoStack.size());
        }

        return success;
    }

    /**
     * Undo last command
     */
    public boolean undo() {
        if (undoStack.isEmpty()) {
            log.warn("Nothing to undo");
            return false;
        }

        UndoableCommand command = undoStack.pop();
        log.info("Undoing command: {}", command.getName());

        command.undo();
        redoStack.push(command);

        return true;
    }

    /**
     * Redo last undone command
     */
    public boolean redo() {
        if (redoStack.isEmpty()) {
            log.warn("Nothing to redo");
            return false;
        }

        UndoableCommand command = redoStack.pop();
        log.info("Redoing command: {}", command.getName());

        command.execute();
        undoStack.push(command);

        return true;
    }

    /**
     * Check if undo is available
     */
    public boolean canUndo() {
        return !undoStack.isEmpty();
    }

    /**
     * Check if redo is available
     */
    public boolean canRedo() {
        return !redoStack.isEmpty();
    }

    /**
     * Get undo description
     */
    public String getUndoDescription() {
        if (undoStack.isEmpty()) {
            return null;
        }
        return undoStack.peek().getUndoDescription();
    }

    /**
     * Clear history
     */
    public void clear() {
        undoStack.clear();
        redoStack.clear();
        log.info("Command history cleared");
    }

    /**
     * Get history size
     */
    public int getHistorySize() {
        return undoStack.size();
    }
}
```

---

## 💻 Implementation in Node.js

### Scenario 1: Background Job Queue with Bull

```typescript
// command.ts - Command interface
export interface Command {
  execute(): Promise<boolean>;
  undo?(): Promise<void>;
  getName(): string;
  getMetadata(): CommandMetadata;
}

export interface CommandMetadata {
  commandId: string;
  commandType: string;
  createdAt: Date;
  retryCount: number;
  maxRetries: number;
  userId?: string;
  tenantId?: string;
}
```

```typescript
// sendEmailCommand.ts
import { Command, CommandMetadata } from "./command";
import { EmailService } from "../services/emailService";

export class SendEmailCommand implements Command {
  constructor(
    private emailService: EmailService,
    private to: string,
    private subject: string,
    private body: string,
    private metadata: CommandMetadata
  ) {}

  async execute(): Promise<boolean> {
    console.log(`📧 Executing SendEmailCommand: ${this.metadata.commandId}`);

    try {
      await this.emailService.sendEmail(this.to, this.subject, this.body);
      console.log(`✅ Email sent successfully: ${this.metadata.commandId}`);
      return true;
    } catch (error) {
      console.error(
        `❌ Failed to send email: ${this.metadata.commandId}`,
        error
      );
      return false;
    }
  }

  async undo(): Promise<void> {
    console.warn("Cannot undo email sending");
  }

  getName(): string {
    return "SendEmail";
  }

  getMetadata(): CommandMetadata {
    return this.metadata;
  }
}
```

```typescript
// generateReportCommand.ts
import { Command, CommandMetadata } from "./command";
import { ReportService } from "../services/reportService";

export class GenerateReportCommand implements Command {
  private generatedReportPath?: string;

  constructor(
    private reportService: ReportService,
    private reportType: string,
    private userId: string,
    private metadata: CommandMetadata
  ) {}

  async execute(): Promise<boolean> {
    console.log(
      `📊 Executing GenerateReportCommand: ${this.metadata.commandId}`
    );

    try {
      this.generatedReportPath = await this.reportService.generateReport(
        this.reportType,
        this.userId
      );
      console.log(`✅ Report generated: ${this.generatedReportPath}`);
      return true;
    } catch (error) {
      console.error(
        `❌ Failed to generate report: ${this.metadata.commandId}`,
        error
      );
      return false;
    }
  }

  async undo(): Promise<void> {
    if (this.generatedReportPath) {
      console.log(`🗑️  Deleting generated report: ${this.generatedReportPath}`);
      await this.reportService.deleteReport(this.generatedReportPath);
    }
  }

  getName(): string {
    return "GenerateReport";
  }

  getMetadata(): CommandMetadata {
    return this.metadata;
  }
}
```

```typescript
// commandQueue.ts - Using Bull for queue management
import Bull, { Job, Queue } from 'bull';
import { Command } from './command';
import Redis from 'ioredis';

export class CommandQueue {
  private queue: Queue;

  constructor(redisUrl: string) {
    this.queue = new Bull('commands', {
      redis: redisUrl,
      defaultJobOptions: {
        attempts: 3,
        backoff: {
          type: 'exponential',
          delay: 2000, // Start with 2 seconds
        },
        removeOnComplete: 100, // Keep last 100 completed jobs
        removeOnFail: false, // Keep failed jobs for inspection
      },
    });

    this.setupProcessor();
    this.setupEventHandlers();
  }

  private setupProcessor(): void {
this.queue.process(async (job: Job) => {
console.log(⚙️  Processing job: ${job.id} (${job.name}));
  const command: Command = job.data.command;

  try {
    const success = await command.execute();

    if (!success) {
      throw new Error(`Command execution returned false`);
    }

    console.log(`✅ Job completed: ${job.id}`);
    return { success: true, commandId: command.getMetadata().commandId };
  } catch (error) {
    console.error(`❌ Job failed: ${job.id}`, error);
    throw error; // Let Bull handle retry
  }
});
}
private setupEventHandlers(): void {
this.queue.on('completed', (job, result) => {
console.log(✅ Job ${job.id} completed:, result);
});
this.queue.on('failed', (job, error) => {
  console.error(`❌ Job ${job.id} failed:`, error.message);
});

this.queue.on('stalled', (job) => {
  console.warn(`⚠️  Job ${job.id} stalled`);
});
}
/**

Add command to queue
*/
async enqueue(command: Command, priority?: number): Promise<string> {
console.log(➕ Enqueuing command: ${command.getName()});

const job = await this.queue.add(
  command.getName(),
  { command },
  {
    priority: priority || 0,
    jobId: command.getMetadata().commandId,
  }
);

return job.id.toString();
}
/**

Get queue statistics
*/
async getStats(): Promise<{
waiting: number;
active: number;
completed: number;
failed: number;
delayed: number;
}> {
const [waiting, active, completed, failed, delayed] = await Promise.all([
this.queue.getWaitingCount(),
this.queue.getActiveCount(),
this.queue.getCompletedCount(),
this.queue.getFailedCount(),
this.queue.getDelayedCount(),
]);

return { waiting, active, completed, failed, delayed };
}
/**

Get failed jobs
*/
async getFailedJobs(): Promise<Job[]> {
return this.queue.getFailed();
}

/**

Retry failed job
*/
async retryFailedJob(jobId: string): Promise<void> {
const job = await this.queue.getJob(jobId);
if (job) {
await job.retry();
console.log(🔄 Retrying job: ${jobId});
}
}

/**

Clean old jobs
*/
async cleanOldJobs(grace: number = 7 * 24 * 60 * 60 * 1000): Promise<void> {
await this.queue.clean(grace, 'completed');
await this.queue.clean(grace, 'failed');
console.log(🧹 Cleaned jobs older than ${grace}ms);
}

/**

Close queue
*/
async close(): Promise<void> {
await this.queue.close();
console.log('👋 Command queue closed');
}
}

// commandService.ts - Service layer
import { v4 as uuidv4 } from 'uuid';
import { CommandQueue } from './commandQueue';
import { SendEmailCommand } from './sendEmailCommand';
import { GenerateReportCommand } from './generateReportCommand';
import { EmailService } from '../services/emailService';
import { ReportService } from '../services/reportService';
import { CommandMetadata } from './command';

export class CommandService {
  constructor(
    private commandQueue: CommandQueue,
    private emailService: EmailService,
    private reportService: ReportService
  ) {}

  /**
   * Schedule email sending
   */
  async scheduleEmail(
    to: string,
    subject: string,
    body: string,
    userId: string
  ): Promise<string> {
    console.log(`📧 Scheduling email to: ${to}`);

    const metadata = this.createMetadata('SendEmail', userId);

    const command = new SendEmailCommand(
      this.emailService,
      to,
      subject,
      body,
      metadata
    );

    return this.commandQueue.enqueue(command);
  }

  /**
   * Schedule report generation
   */
  async scheduleReportGeneration(
    reportType: string,
    userId: string,
    priority?: number
  ): Promise<string> {
    console.log(`📊 Scheduling report generation: ${reportType}`);

    const metadata = this.createMetadata('GenerateReport', userId);

    const command = new GenerateReportCommand(
      this.reportService,
      reportType,
      userId,
      metadata
    );

    return this.commandQueue.enqueue(command, priority);
  }

  /**
   * Get queue statistics
   */
  async getQueueStats() {
    return this.commandQueue.getStats();
  }

  /**
   * Get failed jobs
   */
  async getFailedJobs() {
    return this.commandQueue.getFailedJobs();
  }

  /**
   * Retry failed job
   */
  async retryJob(jobId: string): Promise<void> {
    return this.commandQueue.retryFailedJob(jobId);
  }

  private createMetadata(commandType: string, userId: string): CommandMetadata {
    return {
      commandId: uuidv4(),
      commandType,
      createdAt: new Date(),
      retryCount: 0,
      maxRetries: 3,
      userId,
    };
  }
}

```

```typescript
// jobController.ts - Express controller
import { Request, Response } from "express";
import { CommandService } from "./commandService";

export class JobController {
  constructor(private commandService: CommandService) {}

  async scheduleEmail(req: Request, res: Response): Promise<void> {
    try {
      const { to, subject, body, userId } = req.body;

      const jobId = await this.commandService.scheduleEmail(
        to,
        subject,
        body,
        userId
      );

      res.json({
        success: true,
        jobId,
        message: "Email job scheduled",
      });
    } catch (error) {
      res.status(500).json({
        success: false,
        error: error.message,
      });
    }
  }

  async scheduleReport(req: Request, res: Response): Promise<void> {
    try {
      const { reportType, userId, priority } = req.body;

      const jobId = await this.commandService.scheduleReportGeneration(
        reportType,
        userId,
        priority
      );

      res.json({
        success: true,
        jobId,
        message: "Report job scheduled",
      });
    } catch (error) {
      res.status(500).json({
        success: false,
        error: error.message,
      });
    }
  }

  async getStats(req: Request, res: Response): Promise<void> {
    try {
      const stats = await this.commandService.getQueueStats();
      res.json(stats);
    } catch (error) {
      res.status(500).json({
        success: false,
        error: error.message,
      });
    }
  }

  async getFailedJobs(req: Request, res: Response): Promise<void> {
    try {
      const failedJobs = await this.commandService.getFailedJobs();
      res.json({
        count: failedJobs.length,
        jobs: failedJobs.map((job) => ({
          id: job.id,
          name: job.name,
          failedReason: job.failedReason,
          attemptsMade: job.attemptsMade,
          timestamp: job.timestamp,
        })),
      });
    } catch (error) {
      res.status(500).json({
        success: false,
        error: error.message,
      });
    }
  }

  async retryJob(req: Request, res: Response): Promise<void> {
    try {
      const { jobId } = req.params;

      await this.commandService.retryJob(jobId);

      res.json({
        success: true,
        message: "Job retry initiated",
      });
    } catch (error) {
      res.status(500).json({
        success: false,
        error: error.message,
      });
    }
  }
}
```

### Scenario 2: Undo/Redo System

```typescript
// undoableCommand.ts
import { Command } from "./command";

export interface UndoableCommand extends Command {
  undo(): Promise<void>;
  canUndo(): boolean;
  getUndoDescription(): string;
}
```

```typescript
// updateUserCommand.ts
import { UndoableCommand } from "./undoableCommand";
import { CommandMetadata } from "./command";
import { UserRepository } from "../repositories/userRepository";

export class UpdateUserCommand implements UndoableCommand {
  private oldName?: string;
  private oldEmail?: string;
  private executed = false;

  constructor(
    private userRepository: UserRepository,
    private userId: string,
    private newName: string,
    private newEmail: string,
    private metadata: CommandMetadata
  ) {}

  async execute(): Promise<boolean> {
    console.log(`✏️  Executing UpdateUserCommand: ${this.userId}`);

    try {
      const user = await this.userRepository.findById(this.userId);

      if (!user) {
        throw new Error("User not found");
      }

      // Store old values for undo
      this.oldName = user.name;
      this.oldEmail = user.email;

      // Update user
      user.name = this.newName;
      user.email = this.newEmail;
      await this.userRepository.save(user);

      this.executed = true;
      console.log(`✅ User updated: ${this.userId}`);
      return true;
    } catch (error) {
      console.error(`❌ Failed to update user: ${this.userId}`, error);
      return false;
    }
  }

  async undo(): Promise<void> {
    if (!this.executed) {
      console.warn("Cannot undo: command not executed");
      return;
    }

    console.log(`↩️  Undoing UpdateUserCommand: ${this.userId}`);

    try {
      const user = await this.userRepository.findById(this.userId);

      if (!user) {
        throw new Error("User not found");
      }

      // Restore old values
      user.name = this.oldName!;
      user.email = this.oldEmail!;
      await this.userRepository.save(user);

      console.log(`✅ User update undone: ${this.userId}`);
    } catch (error) {
      console.error(`❌ Failed to undo user update: ${this.userId}`, error);
    }
  }

  canUndo(): boolean {
    return this.executed;
  }

  getUndoDescription(): string {
    return `Undo user update: ${this.newName} -> ${this.oldName}`;
  }

  getName(): string {
    return "UpdateUser";
  }

  getMetadata(): CommandMetadata {
    return this.metadata;
  }
}
```

```typescript
// commandHistory.ts - Undo/Redo manager
import { UndoableCommand } from "./undoableCommand";

export class CommandHistory {
  private undoStack: UndoableCommand[] = [];
  private redoStack: UndoableCommand[] = [];

  constructor(private maxHistorySize: number = 50) {}

  /**
   * Execute command and add to history
   */
  async executeCommand(command: UndoableCommand): Promise<boolean> {
    console.log(`▶️  Executing command: ${command.getName()}`);

    const success = await command.execute();

    if (success && command.canUndo()) {
      this.undoStack.push(command);
      this.redoStack = []; // Clear redo stack on new command

      // Limit history size
      if (this.undoStack.length > this.maxHistorySize) {
        this.undoStack.shift();
      }

      console.log(
        `📚 Command added to history (size: ${this.undoStack.length})`
      );
    }

    return success;
  }

  /**
   * Undo last command
   */
  async undo(): Promise<boolean> {
    if (this.undoStack.length === 0) {
      console.warn("Nothing to undo");
      return false;
    }

    const command = this.undoStack.pop()!;
    console.log(`↩️  Undoing command: ${command.getName()}`);

    await command.undo();
    this.redoStack.push(command);

    return true;
  }

  /**
   * Redo last undone command
   */
  async redo(): Promise<boolean> {
    if (this.redoStack.length === 0) {
      console.warn("Nothing to redo");
      return false;
    }

    const command = this.redoStack.pop()!;
    console.log(`↪️  Redoing command: ${command.getName()}`);

    await command.execute();
    this.undoStack.push(command);

    return true;
  }

  /**
   * Check if undo is available
   */
  canUndo(): boolean {
    return this.undoStack.length > 0;
  }

  /**
   * Check if redo is available
   */
  canRedo(): boolean {
    return this.redoStack.length > 0;
  }

  /**
   * Get undo description
   */
  getUndoDescription(): string | null {
    if (this.undoStack.length === 0) {
      return null;
    }
    return this.undoStack[this.undoStack.length - 1].getUndoDescription();
  }

  /**
   * Get redo description
   */
  getRedoDescription(): string | null {
    if (this.redoStack.length === 0) {
      return null;
    }
    return this.redoStack[this.redoStack.length - 1].getUndoDescription();
  }

  /**
   * Clear history
   */
  clear(): void {
    this.undoStack = [];
    this.redoStack = [];
    console.log("📚 Command history cleared");
  }

  /**
   * Get history info
   */
  getHistoryInfo(): {
    undoCount: number;
    redoCount: number;
    canUndo: boolean;
    canRedo: boolean;
    undoDescription: string | null;
    redoDescription: string | null;
  } {
    return {
      undoCount: this.undoStack.length,
      redoCount: this.redoStack.length,
      canUndo: this.canUndo(),
      canRedo: this.canRedo(),
      undoDescription: this.getUndoDescription(),
      redoDescription: this.getRedoDescription(),
    };
  }
}
```

```typescript
// Usage example
import { CommandHistory } from "./commandHistory";
import { UpdateUserCommand } from "./updateUserCommand";
import { UserRepository } from "../repositories/userRepository";

const history = new CommandHistory(50);
const userRepo = new UserRepository();

// Execute a command
const command1 = new UpdateUserCommand(
  userRepo,
  "user123",
  "John Doe",
  "john@example.com",
  metadata
);

await history.executeCommand(command1);

// Execute another command
const command2 = new UpdateUserCommand(
  userRepo,
  "user123",
  "Jane Doe",
  "jane@example.com",
  metadata
);

await history.executeCommand(command2);

// Undo last command
await history.undo(); // Reverts to "John Doe"

// Redo
await history.redo(); // Changes back to "Jane Doe"

// Get history info
const info = history.getHistoryInfo();
console.log("Undo stack size:", info.undoCount);
console.log("Can undo:", info.canUndo);
console.log("Next undo:", info.undoDescription);
```

---

## 💻 Implementation in Go

### Scenario 1: Background Job Queue

```go
// command.go - Command interface
package command

import "time"

// Command interface
type Command interface {
    Execute() error
    Undo() error
    GetName() string
    GetMetadata() *CommandMetadata
}

// CommandMetadata holds command information
type CommandMetadata struct {
    CommandID   string
    CommandType string
    CreatedAt   time.Time
    RetryCount  int
    MaxRetries  int
    UserID      string
    TenantID    string
}
```

```go
// send_email_command.go
package command

import (
    "fmt"
    "myapp/service"
)

type SendEmailCommand struct {
    emailService *service.EmailService
    to           string
    subject      string
    body         string
    metadata     *CommandMetadata
}

func NewSendEmailCommand(
    emailService *service.EmailService,
    to, subject, body string,
    metadata *CommandMetadata,
) *SendEmailCommand {
    return &SendEmailCommand{
        emailService: emailService,
        to:           to,
        subject:      subject,
        body:         body,
        metadata:     metadata,
    }
}

func (c *SendEmailCommand) Execute() error {
    fmt.Printf("📧 Executing SendEmailCommand: %s to %s\n",
        c.metadata.CommandID, c.to)

    err := c.emailService.SendEmail(c.to, c.subject, c.body)
    if err != nil {
        fmt.Printf("❌ Failed to send email: %s - %v\n",
            c.metadata.CommandID, err)
        return err
    }

    fmt.Printf("✅ Email sent successfully: %s\n", c.metadata.CommandID)
    return nil
}

func (c *SendEmailCommand) Undo() error {
    fmt.Printf("⚠️  Cannot undo email sending: %s\n", c.metadata.CommandID)
    return nil
}

func (c *SendEmailCommand) GetName() string {
    return "SendEmail"
}

func (c *SendEmailCommand) GetMetadata() *CommandMetadata {
    return c.metadata
}
```

```go
// generate_report_command.go
package command

import (
    "fmt"
    "myapp/service"
)

type GenerateReportCommand struct {
    reportService      *service.ReportService
    reportType         string
    userID             string
    metadata           *CommandMetadata
    generatedReportPath string
}

func NewGenerateReportCommand(
    reportService *service.ReportService,
    reportType, userID string,
    metadata *CommandMetadata,
) *GenerateReportCommand {
    return &GenerateReportCommand{
        reportService: reportService,
        reportType:    reportType,
        userID:        userID,
        metadata:      metadata,
    }
}

func (c *GenerateReportCommand) Execute() error {
    fmt.Printf("📊 Executing GenerateReportCommand: %s type: %s\n",
        c.metadata.CommandID, c.reportType)

    reportPath, err := c.reportService.GenerateReport(c.reportType, c.userID)
    if err != nil {
        fmt.Printf("❌ Failed to generate report: %s - %v\n",
            c.metadata.CommandID, err)
        return err
    }

    c.generatedReportPath = reportPath
    fmt.Printf("✅ Report generated: %s at %s\n",
        c.metadata.CommandID, reportPath)
    return nil
}

func (c *GenerateReportCommand) Undo() error {
    if c.generatedReportPath != "" {
        fmt.Printf("🗑️  Deleting generated report: %s\n", c.generatedReportPath)
        return c.reportService.DeleteReport(c.generatedReportPath)
    }
    return nil
}

func (c *GenerateReportCommand) GetName() string {
    return "GenerateReport"
}

func (c *GenerateReportCommand) GetMetadata() *CommandMetadata {
    return c.metadata
}
```

```go
// command_queue.go - Command queue with worker pool
package command

import (
    "fmt"
    "sync"
    "time"
)

type CommandQueue struct {
    queue          chan Command
    workers        int
    failedCommands map[string]Command
    mu             sync.RWMutex
    wg             sync.WaitGroup
    stop           chan struct{}
}

func NewCommandQueue(workers, queueSize int) *CommandQueue {
    cq := &CommandQueue{
        queue:          make(chan Command, queueSize),
        workers:        workers,
        failedCommands: make(map[string]Command),
        stop:           make(chan struct{}),
    }

    // Start worker pool
    cq.startWorkers()

    // Start retry scheduler
    go cq.retryScheduler()

    return cq
}

func (cq *CommandQueue) startWorkers() {
    for i := 0; i < cq.workers; i++ {
        cq.wg.Add(1)
        go cq.worker(i)
    }
}

func (cq *CommandQueue) worker(id int) {
    defer cq.wg.Done()

    fmt.Printf("👷 Worker %d started\n", id)

    for {
        select {
        case cmd := <-cq.queue:
            cq.processCommand(cmd, id)
        case <-cq.stop:
            fmt.Printf("👷 Worker %d stopped\n", id)
            return
        }
    }
}

func (cq *CommandQueue) processCommand(cmd Command, workerID int) {
    metadata := cmd.GetMetadata()

    fmt.Printf("⚙️  Worker %d processing: %s (%s)\n",
        workerID, cmd.GetName(), metadata.CommandID)

    err := cmd.Execute()

    if err != nil {
        cq.handleFailedCommand(cmd, err)
    } else {
        // Remove from failed commands if it was there
        cq.mu.Lock()
        delete(cq.failedCommands, metadata.CommandID)
        cq.mu.Unlock()

        fmt.Printf("✅ Worker %d completed: %s\n", workerID, metadata.CommandID)
    }
}

func (cq *CommandQueue) handleFailedCommand(cmd Command, err error) {
    metadata := cmd.GetMetadata()

    if metadata.RetryCount < metadata.MaxRetries {
        fmt.Printf("⚠️  Command failed, will retry: %s (attempt %d/%d)\n",
            metadata.CommandID, metadata.RetryCount+1, metadata.MaxRetries)

        // Increment retry count
        metadata.RetryCount++

        // Store for retry
        cq.mu.Lock()
        cq.failedCommands[metadata.CommandID] = cmd
        cq.mu.Unlock()
    } else {
        fmt.Printf("❌ Command failed permanently after %d retries: %s\n",
            metadata.MaxRetries, metadata.CommandID)

        // Move to dead letter queue
        cq.moveToDeadLetterQueue(cmd)
    }
}

func (cq *CommandQueue) retryScheduler() {
    ticker := time.NewTicker(60 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            cq.retryFailedCommands()
        case <-cq.stop:
            return
        }
    }
}

func (cq *CommandQueue) retryFailedCommands() {
    cq.mu.Lock()
    if len(cq.failedCommands) == 0 {
        cq.mu.Unlock()
        return
    }

    fmt.Printf("🔄 Retrying %d failed commands\n", len(cq.failedCommands))

    // Copy commands to retry
    commandsToRetry := make([]Command, 0, len(cq.failedCommands))
    for _, cmd := range cq.failedCommands {
        commandsToRetry = append(commandsToRetry, cmd)
    }

    // Clear failed commands map
    cq.failedCommands = make(map[string]Command)
    cq.mu.Unlock()

    // Re-enqueue commands
    for _, cmd := range commandsToRetry {
        cq.Enqueue(cmd)
    }
}

func (cq *CommandQueue) moveToDeadLetterQueue(cmd Command) {
    fmt.Printf("💀 Moving command to dead letter queue: %s\n",
        cmd.GetMetadata().CommandID)

    // Store in database for manual inspection/retry
    // Implementation depends on your persistence layer
}

func (cq *CommandQueue) Enqueue(cmd Command) {
    fmt.Printf("➕ Enqueuing command: %s (%s)\n",
        cmd.GetName(), cmd.GetMetadata().CommandID)

    select {
    case cq.queue <- cmd:
        // Successfully enqueued
    default:
        fmt.Printf("⚠️  Queue full, dropping command: %s\n",
            cmd.GetMetadata().CommandID)
    }
}

func (cq *CommandQueue) GetStats() QueueStats {
    cq.mu.RLock()
    defer cq.mu.RUnlock()

    return QueueStats{
        QueueSize:           len(cq.queue),
        FailedCommandsCount: len(cq.failedCommands),
    }
}

func (cq *CommandQueue) Close() {
    fmt.Println("🛑 Shutting down command queue")
    close(cq.stop)
    cq.wg.Wait()
    close(cq.queue)
}

type QueueStats struct {
    QueueSize           int
    FailedCommandsCount int
}
```

```go
// command_service.go - Service layer
package service

import (
    "fmt"
    "myapp/command"
    "time"
    "github.com/google/uuid"
)

type CommandService struct {
    commandQueue  *command.CommandQueue
    emailService  *EmailService
    reportService *ReportService
}

func NewCommandService(
    commandQueue *command.CommandQueue,
    emailService *EmailService,
    reportService *ReportService,
) *CommandService {
    return &CommandService{
        commandQueue:  commandQueue,
        emailService:  emailService,
        reportService: reportService,
    }
}

func (s *CommandService) ScheduleEmail(
    to, subject, body, userID string,
) string {
    fmt.Printf("📧 Scheduling email to: %s\n", to)

    metadata := s.createMetadata("SendEmail", userID)

    cmd := command.NewSendEmailCommand(
        s.emailService,
        to,
        subject,
        body,
        metadata,
    )

    s.commandQueue.Enqueue(cmd)

    return metadata.CommandID
}

func (s *CommandService) ScheduleReportGeneration(
    reportType, userID string,
) string {
    fmt.Printf("📊 Scheduling report generation: %s\n", reportType)

    metadata := s.createMetadata("GenerateReport", userID)

    cmd := command.NewGenerateReportCommand(
        s.reportService,
        reportType,
        userID,
        metadata,
    )

    s.commandQueue.Enqueue(cmd)

    return metadata.CommandID
}

func (s *CommandService) GetQueueStats() command.QueueStats {
    return s.commandQueue.GetStats()
}

func (s *CommandService) createMetadata(commandType, userID string) *command.CommandMetadata {
    return &command.CommandMetadata{
        CommandID:   uuid.New().String(),
        CommandType: commandType,
        CreatedAt:   time.Now(),
        RetryCount:  0,
        MaxRetries:  3,
        UserID:      userID,
    }
}
```

```go
// handler.go - HTTP handler
package handler

import (
    "encoding/json"
    "myapp/service"
    "net/http"
)

type JobHandler struct {
    commandService *service.CommandService
}

func NewJobHandler(commandService *service.CommandService) *JobHandler {
    return &JobHandler{
        commandService: commandService,
    }
}

func (h *JobHandler) ScheduleEmail(w http.ResponseWriter, r *http.Request) {
    var req struct {
        To      string `json:"to"`
        Subject string `json:"subject"`
        Body    string `json:"body"`
        UserID  string `json:"userId"`
    }

    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    jobID := h.commandService.ScheduleEmail(
        req.To,
        req.Subject,
        req.Body,
        req.UserID,
    )

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]interface{}{
        "success": true,
        "jobId":   jobID,
        "message": "Email job scheduled",
    })
}

func (h *JobHandler) ScheduleReport(w http.ResponseWriter, r *http.Request) {
    var req struct {
        ReportType string `json:"reportType"`
        UserID     string `json:"userId"`
    }

    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    jobID := h.commandService.ScheduleReportGeneration(
        req.ReportType,
        req.UserID,
    )

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]interface{}{
        "success": true,
        "jobId":   jobID,
        "message": "Report job scheduled",
    })
}

func (h *JobHandler) GetStats(w http.ResponseWriter, r *http.Request) {
    stats := h.commandService.GetQueueStats()

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(stats)
}
```

### Scenario 2: Undo/Redo System

```go
// undoable_command.go
package command

// UndoableCommand extends Command with undo support
type UndoableCommand interface {
    Command
    CanUndo() bool
    GetUndoDescription() string
}
```

```go
// update_user_command.go
package command

import (
    "fmt"
    "myapp/model"
    "myapp/repository"
)

type UpdateUserCommand struct {
    userRepo *repository.UserRepository
    userID   string
    newName  string
    newEmail string
    metadata *CommandMetadata

    // Store old values for undo
    oldName  string
    oldEmail string
    executed bool
}

func NewUpdateUserCommand(
    userRepo *repository.UserRepository,
    userID, newName, newEmail string,
    metadata *CommandMetadata,
) *UpdateUserCommand {
    return &UpdateUserCommand{
        userRepo: userRepo,
        userID:   userID,
        newName:  newName,
        newEmail: newEmail,
        metadata: metadata,
    }
}

func (c *UpdateUserCommand) Execute() error {
    fmt.Printf("✏️  Executing UpdateUserCommand: %s\n", c.userID)

    user, err := c.userRepo.FindByID(c.userID)
    if err != nil {
        return fmt.Errorf("user not found: %w", err)
    }

    // Store old values
    c.oldName = user.Name
    c.oldEmail = user.Email

    // Update user
    user.Name = c.newName
    user.Email = c.newEmail

    err = c.userRepo.Save(user)
    if err != nil {
        return fmt.Errorf("failed to save user: %w", err)
    }

    c.executed = true
    fmt.Printf("✅ User updated: %s\n", c.userID)
    return nil
}

func (c *UpdateUserCommand) Undo() error {
    if !c.executed {
        return fmt.Errorf("cannot undo: command not executed")
    }

    fmt.Printf("↩️  Undoing UpdateUserCommand: %s\n", c.userID)

    user, err := c.userRepo.FindByID(c.userID)
    if err != nil {
        return fmt.Errorf("user not found: %w", err)
    }

    // Restore old values
    user.Name = c.oldName
    user.Email = c.oldEmail

    err = c.userRepo.Save(user)
    if err != nil {
        return fmt.Errorf("failed to undo: %w", err)
    }

    fmt.Printf("✅ User update undone: %s\n", c.userID)
    return nil
}

func (c *UpdateUserCommand) CanUndo() bool {
    return c.executed
}

func (c *UpdateUserCommand) GetUndoDescription() string {
    return fmt.Sprintf("Undo user update: %s -> %s", c.newName, c.oldName)
}

func (c *UpdateUserCommand) GetName() string {
    return "UpdateUser"
}

func (c *UpdateUserCommand) GetMetadata() *CommandMetadata {
    return c.metadata
}
```

```go
// command_history.go - Undo/Redo manager
package command

import (
    "fmt"
    "sync"
)

type CommandHistory struct {
    undoStack      []UndoableCommand
    redoStack      []UndoableCommand
    maxHistorySize int
    mu             sync.Mutex
}

func NewCommandHistory(maxHistorySize int) *CommandHistory {
    return &CommandHistory{
        undoStack:      make([]UndoableCommand, 0),
        redoStack:      make([]UndoableCommand, 0),
        maxHistorySize: maxHistorySize,
    }
}

// ExecuteCommand executes a command and adds it to history
func (h *CommandHistory) ExecuteCommand(cmd UndoableCommand) error {
    h.mu.Lock()
    defer h.mu.Unlock()

    fmt.Printf("▶️  Executing command: %s\n", cmd.GetName())

    err := cmd.Execute()
    if err != nil {
        return err
    }

    if cmd.CanUndo() {
        h.undoStack = append(h.undoStack, cmd)
        h.redoStack = nil // Clear redo stack on new command

        // Limit history size
        if len(h.undoStack) > h.maxHistorySize {
            h.undoStack = h.undoStack[1:]
        }

        fmt.Printf("📚 Command added to history (size: %d)\n", len(h.undoStack))
    }

    return nil
}

// Undo undoes the last command
func (h *CommandHistory) Undo() error {
    h.mu.Lock()
    defer h.mu.Unlock()

    if len(h.undoStack) == 0 {
        return fmt.Errorf("nothing to undo")
    }

    // Pop from undo stack
    lastIndex := len(h.undoStack) - 1
    cmd := h.undoStack[lastIndex]
    h.undoStack = h.undoStack[:lastIndex]

    fmt.Printf("↩️  Undoing command: %s\n", cmd.GetName())

    err := cmd.Undo()
    if err != nil {
        return err
    }

    // Push to redo stack
    h.redoStack = append(h.redoStack, cmd)

    return nil
}

// Redo redoes the last undone command
func (h *CommandHistory) Redo() error {
    h.mu.Lock()
    defer h.mu.Unlock()

    if len(h.redoStack) == 0 {
        return fmt.Errorf("nothing to redo")
    }

    // Pop from redo stack
    lastIndex := len(h.redoStack) - 1
    cmd := h.redoStack[lastIndex]
    h.redoStack = h.redoStack[:lastIndex]

    fmt.Printf("↪️  Redoing command: %s\n", cmd.GetName())

    err := cmd.Execute()
    if err != nil {
        return err
    }

    // Push to undo stack
    h.undoStack = append(h.undoStack, cmd)

    return nil
}

// CanUndo checks if undo is available
func (h *CommandHistory) CanUndo() bool {
    h.mu.Lock()
    defer h.mu.Unlock()
    return len(h.undoStack) > 0
}

// CanRedo checks if redo is available
func (h *CommandHistory) CanRedo() bool {
    h.mu.Lock()
    defer h.mu.Unlock()
    return len(h.redoStack) > 0
}

// GetUndoDescription gets description of next undo
func (h *CommandHistory) GetUndoDescription() string {
    h.mu.Lock()
    defer h.mu.Unlock()

    if len(h.undoStack) == 0 {
        return ""
    }

    return h.undoStack[len(h.undoStack)-1].GetUndoDescription()
}

// GetRedoDescription gets description of next redo
func (h *CommandHistory) GetRedoDescription() string {
    h.mu.Lock()
    defer h.mu.Unlock()

    if len(h.redoStack) == 0 {
        return ""
    }

    return h.redoStack[len(h.redoStack)-1].GetUndoDescription()
}

// Clear clears all history
func (h *CommandHistory) Clear() {
    h.mu.Lock()
    defer h.mu.Unlock()

    h.undoStack = nil
    h.redoStack = nil
    fmt.Println("📚 Command history cleared")
}

// GetHistorySize returns the number of commands in history
func (h *CommandHistory) GetHistorySize() int {
    h.mu.Lock()
    defer h.mu.Unlock()
    return len(h.undoStack)
}

// GetHistoryInfo returns detailed history information
func (h *CommandHistory) GetHistoryInfo() HistoryInfo {
    h.mu.Lock()
    defer h.mu.Unlock()

    return HistoryInfo{
        UndoCount:         len(h.undoStack),
        RedoCount:         len(h.redoStack),
        CanUndo:           len(h.undoStack) > 0,
        CanRedo:           len(h.redoStack) > 0,
        UndoDescription:   h.getUndoDescriptionUnsafe(),
        RedoDescription:   h.getRedoDescriptionUnsafe(),
    }
}

func (h *CommandHistory) getUndoDescriptionUnsafe() string {
    if len(h.undoStack) == 0 {
        return ""
    }
    return h.undoStack[len(h.undoStack)-1].GetUndoDescription()
}

func (h *CommandHistory) getRedoDescriptionUnsafe() string {
    if len(h.redoStack) == 0 {
        return ""
    }
    return h.redoStack[len(h.redoStack)-1].GetUndoDescription()
}

type HistoryInfo struct {
    UndoCount       int    `json:"undoCount"`
    RedoCount       int    `json:"redoCount"`
    CanUndo         bool   `json:"canUndo"`
    CanRedo         bool   `json:"canRedo"`
    UndoDescription string `json:"undoDescription"`
    RedoDescription string `json:"redoDescription"`
}
```

```go
// main.go - Usage example
package main

import (
    "fmt"
    "myapp/command"
    "myapp/repository"
)

func main() {
    // Create command history
    history := command.NewCommandHistory(50)

    // Create repository
    userRepo := repository.NewUserRepository()

    // Create metadata
    metadata := &command.CommandMetadata{
        CommandID:   "cmd-1",
        CommandType: "UpdateUser",
        UserID:      "user-123",
    }

    // Execute first command
    cmd1 := command.NewUpdateUserCommand(
        userRepo,
        "user-123",
        "John Doe",
        "john@example.com",
        metadata,
    )

    err := history.ExecuteCommand(cmd1)
    if err != nil {
        panic(err)
    }

    // Execute second command
    cmd2 := command.NewUpdateUserCommand(
        userRepo,
        "user-123",
        "Jane Doe",
        "jane@example.com",
        metadata,
    )

    err = history.ExecuteCommand(cmd2)
    if err != nil {
        panic(err)
    }

    // Get history info
    info := history.GetHistoryInfo()
    fmt.Printf("History: %+v\n", info)

    // Undo
    err = history.Undo()
    if err != nil {
        panic(err)
    }

    // Redo
    err = history.Redo()
    if err != nil {
        panic(err)
    }
}
```

---

## ✅ Best Practices

### 1. **Keep Commands Self-Contained**

Commands should have all information needed to execute.

```java
// ✅ GOOD - Self-contained
public class SendEmailCommand implements Command {
    private final String to;
    private final String subject;
    private final String body;
    private final EmailService emailService;

    // All dependencies passed in constructor
}

// ❌ BAD - External dependencies
public class SendEmailCommand implements Command {
    public boolean execute() {
        // Relies on global state or service locator
        EmailService service = ServiceLocator.get(EmailService.class);
    }
}
```

### 2. **Make Commands Immutable**

Once created, command parameters shouldn't change.

```typescript
// ✅ GOOD - Immutable command
class UpdateUserCommand {
  constructor(
    private readonly userId: string,
    private readonly newName: string,
    private readonly newEmail: string
  ) {}
}

// ❌ BAD - Mutable
class UpdateUserCommand {
  userId: string;
  newName: string;
  // Can be changed after creation - dangerous!
}
```

### 3. **Implement Proper Error Handling**

Commands should handle errors gracefully.

```go
// ✅ GOOD - Proper error handling
func (c *SendEmailCommand) Execute() error {
    err := c.emailService.SendEmail(c.to, c.subject, c.body)
    if err != nil {
        return fmt.Errorf("failed to send email to %s: %w", c.to, err)
    }
    return nil
}

// ❌ BAD - Swallows errors
func (c *SendEmailCommand) Execute() error {
    c.emailService.SendEmail(c.to, c.subject, c.body)
    return nil // Always succeeds?
}
```

### 4. **Log Command Execution**

Track what's happening for debugging.

```java
// ✅ GOOD - Comprehensive logging
@Override
public boolean execute() {
    log.info("Executing {}: {}", getName(), metadata.getCommandId());
    try {
        // Execute logic
        log.info("Successfully executed: {}", metadata.getCommandId());
        return true;
    } catch (Exception e) {
        log.error("Failed to execute: {}", metadata.getCommandId(), e);
        return false;
    }
}
```

### 5. **Implement Idempotency**

Commands should be safe to execute multiple times.

```typescript
// ✅ GOOD - Idempotent
async execute(): Promise<boolean> {
  // Check if already processed
  const exists = await this.repo.exists(this.recordId);
  if (exists) {
    console.log('Record already exists, skipping');
    return true;
  }

  await this.repo.create(this.record);
  return true;
}
```

### 6. **Store Undo Information**

For undoable commands, capture state before changes.

```java
// ✅ GOOD - Stores old state
public class UpdateProductCommand implements UndoableCommand {
    private Product oldProduct; // Stored before update

    @Override
    public boolean execute() {
        oldProduct = productRepository.findById(productId);
        // Now can undo
    }
}
```

### 7. **Validate Commands Before Execution**

Check preconditions early.

```typescript
// ✅ GOOD - Validation
async execute(): Promise<boolean> {
  // Validate first
  if (!this.isValid()) {
    throw new Error('Invalid command');
  }

  // Then execute
  await this.performAction();
  return true;
}

private isValid(): boolean {
  return this.email && this.email.includes('@');
}
```

### 8. **Use Command Metadata**

Track important information about commands.

```go
// ✅ GOOD - Rich metadata
type CommandMetadata struct {
    CommandID   string
    CommandType string
    CreatedAt   time.Time
    RetryCount  int
    MaxRetries  int
    UserID      string
    TenantID    string
    Priority    int
    Tags        []string
}
```

### 9. **Implement Command Serialization**

For persistent queues, commands must be serializable.

```java
// ✅ GOOD - Serializable command
@Serializable
public class SendEmailCommand implements Command, Serializable {
    private static final long serialVersionUID = 1L;

    // Transient for non-serializable fields
    private transient EmailService emailService;

    // Serializable state
    private String to;
    private String subject;
    private String body;
}
```

### 10. **Clean Up Resources**

Implement proper cleanup for long-running commands.

```typescript
// ✅ GOOD - Cleanup
class ProcessVideoCommand implements Command {
  async execute(): Promise<boolean> {
    const tempFile = await this.downloadVideo();

    try {
      await this.processVideo(tempFile);
      return true;
    } finally {
      // Always cleanup
      await this.deleteTempFile(tempFile);
    }
  }
}
```

---

## 🚫 Common Pitfalls

### 1. **Commands That Do Too Much**

**Problem**: Commands with multiple responsibilities.

```java
// ❌ PITFALL - God command
public class ProcessOrderCommand implements Command {
    public boolean execute() {
        validateOrder();
        chargePayment();
        updateInventory();
        sendEmail();
        createInvoice();
        updateAnalytics();
        // Too much!
    }
}

// ✅ SOLUTION - Composite command or separate commands
public class ProcessOrderCommand implements Command {
    private final List<Command> steps;

    public boolean execute() {
        for (Command step : steps) {
            if (!step.execute()) {
                rollback();
                return false;
            }
        }
        return true;
    }
}
```

### 2. **Tight Coupling to Concrete Classes**

**Problem**: Commands depend on specific implementations.

```typescript
// ❌ PITFALL
class SendEmailCommand {
  private gmailService = new GmailService(); // Tight coupling!
}

// ✅ SOLUTION - Depend on interfaces
class SendEmailCommand {
  constructor(private emailService: IEmailService) {}
}
```

### 3. **Memory Leaks in Command History**

**Problem**: Unlimited history growth.

```go
// ❌ PITFALL - Unbounded history
type CommandHistory struct {
    history []Command // Grows forever!
}

// ✅ SOLUTION - Limit history size
type CommandHistory struct {
    history        []Command
    maxHistorySize int
}

func (h *CommandHistory) Add(cmd Command) {
    h.history = append(h.history, cmd)
    if len(h.history) > h.maxHistorySize {
        h.history = h.history[1:] // Remove oldest
    }
}
```

### 4. **Not Handling Partial Failures**

**Problem**: No rollback on failure.

```java
// ❌ PITFALL
public boolean execute() {
    step1(); // Succeeds
    step2(); // Fails - step1 not rolled back!
    return false;
}

// ✅ SOLUTION - Transaction pattern
public boolean execute() {
    try {
        beginTransaction();
        step1();
        step2();
        commit();
        return true;
    } catch (Exception e) {
        rollback();
        return false;
    }
}
```

### 5. **Synchronous Execution of Long-Running Commands**

**Problem**: Blocking operations.

```typescript
// ❌ PITFALL - Blocks caller
app.post('/generate-report', (req, res) => {
  const command = new GenerateReportCommand(...);
  command.execute(); // Takes 5 minutes!
  res.json({ success: true });
});

// ✅ SOLUTION - Queue it
app.post('/generate-report', async (req, res) => {
  const command = new GenerateReportCommand(...);
  const jobId = await commandQueue.enqueue(command);
  res.json({ jobId, status: 'queued' });
});
```

### 6. **No Command Priority**

**Problem**: All commands treated equally.

```go
// ❌ PITFALL - FIFO only
type CommandQueue struct {
    queue chan Command
}

// ✅ SOLUTION - Priority queue
type PriorityCommand struct {
    Command
    Priority int
}

type CommandQueue struct {
    queue *PriorityQueue
}
```

### 7. **Undo Stack Corruption**

**Problem**: Undo doesn't properly reverse changes.

```java
// ❌ PITFALL
public void undo() {
    // Assumes current state, doesn't check
    user.setName(oldName);
}

// ✅ SOLUTION - Verify state before undo
public void undo() {
    User current = repository.findById(userId);
    if (!current.getName().equals(newName)) {
        throw new IllegalStateException("State changed, cannot undo");
    }
    current.setName(oldName);
    repository.save(current);
}
```

### 8. **Not Considering Concurrency**

**Problem**: Race conditions in command execution.

```typescript
// ❌ PITFALL - Race condition
class UpdateBalanceCommand {
  async execute() {
    const account = await this.repo.findById(this.accountId);
    account.balance += this.amount;
    await this.repo.save(account); // Lost update!
  }
}

// ✅ SOLUTION - Optimistic locking
class UpdateBalanceCommand {
  async execute() {
    let retries = 3;
    while (retries > 0) {
      try {
        const account = await this.repo.findByIdWithLock(this.accountId);
        account.balance += this.amount;
        account.version++;
        await this.repo.saveWithVersion(account);
        return true;
      } catch (OptimisticLockError) {
        retries--;
      }
    }
    return false;
  }
}
```

### 9. **Command Explosion**

**Problem**: Too many similar command classes.

```java
// ❌ PITFALL - 50+ nearly identical classes
public class SendEmailToUser1Command { }
public class SendEmailToUser2Command { }
// ...

// ✅ SOLUTION - Parameterized command
public class SendEmailCommand {
    private final String userId;
    private final String template;

    public SendEmailCommand(String userId, String template) {
        this.userId = userId;
        this.template = template;
    }
}
```

### 10. **Not Testing Commands**

**Problem**: No unit tests for commands.

```typescript
// ✅ SOLUTION - Test commands
describe('SendEmailCommand', () => {
  it('should send email successfully', async () => {
    const mockEmailService = {
      sendEmail: jest.fn().mockResolvedValue(true)
    };

    const command = new SendEmailCommand(
      mockEmailService,
      'test@example.com',
      'Subject',
      'Body',
      metadata
    );

    const result = await command.execute();

    expect(result).toBe(true);
    expect(mockEmailService.sendEmail).toHaveBeenCalledWith(
      'test@example.com',
      'Subject',
      'Body'
    );
  });

  it('should handle errors gracefully', async () => {
    const mockEmailService = {
      sendEmail: jest.fn().mockRejectedValue(new Error('Network error'))
    };

    const command = new SendEmailCommand(mockEmailService, ...);
    const result = await command.execute();

    expect(result).toBe(false);
  });
});
```

---

## 🎯 Real-World SaaS Use Cases

### 1. **Background Job Processing** ⭐⭐⭐

Email Sending → Queue as SendEmailCommand
Report Generation → Queue as GenerateReportCommand
Data Export → Queue as ExportDataCommand
Image Processing → Queue as ProcessImageCommand

### 2. **Undo/Redo in Document Editors** ⭐⭐⭐

Text Edit → UpdateTextCommand (undoable)
Format Change → FormatCommand (undoable)
Delete → DeleteCommand (undoable)
Insert → InsertCommand (undoable)

### 3. **Multi-Step Wizards** ⭐⭐

Step 1: CreateAccountCommand
Step 2: AddPaymentMethodCommand
Step 3: SelectPlanCommand
Step 4: Complete → Execute all as transaction

### 4. **Audit Logging** ⭐⭐⭐

Every action → Stored as Command
Compliance requirement → Replay commands
Debugging → Review command history

### 5. **API Rate Limiting** ⭐⭐

API Request → Wrapped as Command
Queue → Rate-limited execution
Priority → Premium users get priority queue

### 6. **Scheduled Tasks** ⭐⭐⭐

SendWeeklyReportCommand → Scheduled for Mondays
CleanupOldDataCommand → Scheduled daily
RenewSubscriptionsCommand → Scheduled monthly

### 7. **Distributed Transactions (Saga Pattern)** ⭐⭐

OrderCommand → CreateOrder, ChargePayment, UpdateInventory
Failure → Compensating commands (undo each step)

---

## 📚 Resources for Deeper Understanding

### Books

1. **"Design Patterns: Elements of Reusable Object-Oriented Software"** - Gang of Four

   - Chapter: Behavioral Patterns → Command

2. **"Head First Design Patterns"** - Freeman & Freeman

   - Chapter 6: The Command Pattern (Remote Control example)

3. **"Patterns of Enterprise Application Architecture"** - Martin Fowler

   - Command Query Separation

4. **"Enterprise Integration Patterns"** - Hohpe & Woolf
   - Message Queue patterns

### Online Resources

1. **Refactoring Guru - Command Pattern**

   - https://refactoring.guru/design-patterns/command

2. **Bull Queue Documentation**

   - https://github.com/OptimalBits/bull

3. **Spring Batch Documentation**

   - https://docs.spring.io/spring-batch/reference/html/

4. **RabbitMQ Tutorials**
   - https://www.rabbitmq.com/tutorials

### Articles

1. **"Command Pattern in Modern Applications"** - Baeldung

   - https://www.baeldung.com/java-command-pattern

2. **"Implementing Undo/Redo"** - Martin Fowler

   - Command pattern for undoable operations

3. **"Job Queues in Node.js"**
   - Using Bull and Redis

---

## 🏋️ Practical Exercise

### Exercise: Build a Task Management System with Undo/Redo

**Scenario**: Build a collaborative task management system where users can create, update, delete tasks, and undo/redo their actions.

**Requirements**:

1. **Commands to Implement**:

   - `CreateTaskCommand`: Create new task
   - `UpdateTaskCommand`: Update task details
   - `DeleteTaskCommand`: Delete task (soft delete)
   - `AssignTaskCommand`: Assign task to user
   - `CompleteTaskCommand`: Mark task as complete
   - `AddCommentCommand`: Add comment to task

2. **Features**:

   - Undo/Redo for all commands (unlimited history)
   - Command history per user (isolated)
   - Background job queue for notifications
   - Audit log of all commands
   - Command replay for debugging
   - Macro recording (record sequence of commands)

3. **Advanced**:
   - Priority queue (urgent tasks processed first)
   - Command batching (group related commands)
   - Distributed queue (multiple workers)
   - Failed command retry with exponential backoff
   - Command scheduling (execute at specific time)

**Your Task**:

Implement in your language of choice:

- Command interface and at least 4 concrete commands
- Command queue with retry logic
- Undo/Redo system (CommandHistory)
- REST API for executing commands
- Audit log storage
- Queue monitoring dashboard

**Bonus Challenges**:

- Implement command serialization (persist queue to database)
- Add command validation before execution
- Implement distributed locking for concurrent commands
- Create admin panel to view/replay commands
- Add command analytics (execution time, success rate)
- Implement command snapshots (save system state)

**Acceptance Criteria**:

- ✅ All commands properly execute and can be undone
- ✅ Command queue handles failures gracefully
- ✅ Undo/Redo works correctly with proper state management
- ✅ Audit log captures all command executions
- ✅ Queue can process 1000+ commands/second
- ✅ Failed commands retry with backoff
- ✅ No memory leaks in command history

---

## 📝 Summary

**Command Pattern**:

- ✅ Encapsulates requests as objects
- ✅ Enables queuing, logging, undo/redo
- ✅ Decouples invoker from receiver
- ✅ Perfect for background jobs and transactions
- ✅ Essential for audit trails
- ❌ Can create many command classes
- ❌ Adds indirection overhead
- ❌ Complex for simple operations

**Key Concepts**:

1. **Command**: Encapsulates request as object
2. **Receiver**: Performs actual work
3. **Invoker**: Executes commands
4. **Client**: Creates and configures commands
5. **History**: Manages undo/redo

**When to Use**:

- Background job processing
- Undo/Redo functionality
- Transaction management
- Audit logging
- Request queuing
- Macro recording
- Scheduled tasks

**When NOT to Use**:

- Simple CRUD operations
- Real-time direct actions
- Stateless operations
- No need for queuing/undo

**Remember**: Command pattern turns requests into objects, enabling powerful features like queuing, logging, and undo. It's essential for building robust SaaS applications with background processing and audit trails!

---

## 🔜 What's Next?

Next pattern: **Chain of Responsibility Pattern** - Building request pipelines, middleware chains, and validation flows

Ready to continue? 🚀# Module 2.5: Core Design Patterns for SaaS Applications

---
