# Background Jobs

This document covers the background job system using BullMQ and worker processes.

## Overview

The application uses BullMQ for reliable background job processing. Jobs are queued in Redis and processed by separate worker processes.

## Architecture

### Job Flow

```
API Server
    ↓ (enqueue job)
BullMQ Queue (Redis)
    ↓ (worker picks up)
Worker Process
    ↓ (processes)
Job Processor
    ↓ (completes/fails)
Result stored in Redis
```

### Components

1. **Queue** - Redis-based queue storage
2. **Worker Process** - Separate NestJS process for job execution
3. **Job Processor** - Handles specific job types
4. **Bull Board** - Web UI for monitoring

## Queue Configuration

### Queue Setup

Located in `src/config/bull/bull.config.ts`:

```typescript
export function getConfig(): BullConfig {
  return {
    prefix: `${appPrefix}:bull`,
    redis: redisConfig(),
    defaultJobOptions: {
      removeOnComplete: process.env.QUEUE_REMOVE_ON_COMPLETE === 'true',
      removeOnFail: process.env.QUEUE_REMOVE_ON_FAIL === 'true',
      attempts: parseInt(process.env.QUEUE_FAILED_RETRY_ATTEMPTS || '3'),
      backoff: {
        type: 'exponential',
        delay: 1000,
      },
    },
  };
}
```

### Available Queues

- **Email Queue** - For email sending jobs

## Job Types

### Email Jobs

Located in `src/worker/queues/email/`:

1. **EmailVerification** - Send email verification
2. **SignInMagicLink** - Send magic link
3. **ResetPassword** - Send password reset email

### Job Constants

Located in `src/constants/job.constant.ts`:

```typescript
export enum Job {
  Email = 'email',
}

export enum Queue {
  Email = 'email',
}
```

## Creating Jobs

### Enqueueing Jobs

```typescript
import { InjectQueue } from '@nestjs/bullmq';
import { Queue } from 'bullmq';

@Injectable()
export class EmailService {
  constructor(
    @InjectQueue(Queue.Email)
    private emailQueue: Queue,
  ) {}

  async sendVerificationEmail(userId: string, url: string) {
    await this.emailQueue.add(Job.EmailVerification, {
      userId,
      url,
    });
  }
}
```

### Job Data

```typescript
interface EmailJobData {
  userId: string;
  url: string;
  email?: string;
}
```

## Processing Jobs

### Job Processor

Located in `src/worker/queues/email/email.processor.ts`:

```typescript
@Processor(Queue.Email, {
  concurrency: 1,
  drainDelay: 300,
  stalledInterval: 300000,
  removeOnComplete: {
    age: 86400, // 24 hours
    count: 100,
  },
  limiter: {
    max: 1,
    duration: 150, // 1 job per 150ms
  },
})
export class EmailProcessor extends WorkerHost {
  async process(job: EmailJob, token?: string): Promise<any> {
    switch (job.name) {
      case Job.EmailVerification:
        return await this.emailQueueService.verifyEmail(job.data);
      // ... other cases
    }
  }
}
```

### Processor Options

- **concurrency** - Number of jobs processed simultaneously
- **drainDelay** - Delay before processing next job
- **stalledInterval** - Check for stalled jobs
- **removeOnComplete** - Auto-remove completed jobs
- **limiter** - Rate limiting

## Worker Process

### Worker Module

Located in `src/worker/worker.module.ts`:

```typescript
@Module({
  imports: [EmailQueueModule],
})
export class WorkerModule {}
```

### Starting Worker

Worker runs as separate process:

```bash
# Set IS_WORKER=true
IS_WORKER=true pnpm start

# Or use Docker
docker-compose up worker
```

### Worker vs API Server

- **API Server** (`IS_WORKER=false`) - Handles HTTP requests
- **Worker** (`IS_WORKER=true`) - Processes background jobs

## Monitoring Jobs

### Bull Board

Access Bull Board UI at:
- URL: `http://localhost:3000/api/queues`
- Auth: Basic Auth required

### Bull Board Features

- View all queues
- Monitor job status
- Retry failed jobs
- View job details
- Clean completed/failed jobs

### Job Status

Jobs can have these statuses:
- **waiting** - Queued, waiting to be processed
- **active** - Currently being processed
- **completed** - Successfully completed
- **failed** - Failed after retries
- **delayed** - Scheduled for future execution
- **stalled** - Processing but no progress

## Job Events

### Event Handlers

```typescript
@OnWorkerEvent('active')
async onActive(job: Job) {
  this.logger.debug(`Job ${job.id} is now active`);
}

@OnWorkerEvent('completed')
async onCompleted(job: Job) {
  this.logger.debug(`Job ${job.id} has been completed`);
}

@OnWorkerEvent('failed')
async onFailed(job: Job) {
  this.logger.error(`Job ${job.id} failed: ${job.failedReason}`);
}
```

### Available Events

- `active` - Job started processing
- `progress` - Job progress updated
- `completed` - Job completed successfully
- `failed` - Job failed
- `stalled` - Job stalled
- `error` - Job error occurred

## Error Handling

### Retry Strategy

Jobs automatically retry on failure:

```typescript
defaultJobOptions: {
  attempts: 3, // Retry 3 times
  backoff: {
    type: 'exponential', // Exponential backoff
    delay: 1000, // Start with 1 second
  },
}
```

### Handling Failures

```typescript
@OnWorkerEvent('failed')
async onFailed(job: Job) {
  // Log failure
  this.logger.error(`Job ${job.id} failed: ${job.failedReason}`);

  // Send alert
  // Store failure details
  // Notify administrators
}
```

## Creating New Job Types

### Step 1: Define Job Type

```typescript
// src/constants/job.constant.ts
export enum Job {
  Email = 'email',
  ImageProcessing = 'image-processing', // New job type
}
```

### Step 2: Create Queue Module

```typescript
// src/worker/queues/image-processing/image-processing.module.ts
@Module({
  imports: [BullModule.registerQueue({ name: Queue.ImageProcessing })],
  providers: [ImageProcessingProcessor, ImageProcessingService],
})
export class ImageProcessingQueueModule {}
```

### Step 3: Create Processor

```typescript
// src/worker/queues/image-processing/image-processing.processor.ts
@Processor(Queue.ImageProcessing)
export class ImageProcessingProcessor extends WorkerHost {
  async process(job: ImageProcessingJob): Promise<any> {
    // Process job
  }
}
```

### Step 4: Register Queue

```typescript
// src/worker/worker.module.ts
@Module({
  imports: [
    EmailQueueModule,
    ImageProcessingQueueModule, // Add new queue
  ],
})
export class WorkerModule {}
```

### Step 5: Enqueue Jobs

```typescript
@Injectable()
export class ImageService {
  constructor(
    @InjectQueue(Queue.ImageProcessing)
    private imageQueue: Queue,
  ) {}

  async processImage(imageId: string) {
    await this.imageQueue.add(Job.ImageProcessing, { imageId });
  }
}
```

## Job Scheduling

### Delayed Jobs

```typescript
// Execute after 5 minutes
await this.emailQueue.add(Job.Email, data, {
  delay: 5 * 60 * 1000, // 5 minutes in milliseconds
});
```

### Scheduled Jobs (Cron)

For recurring jobs, use a cron library:

```typescript
import { Cron, CronExpression } from '@nestjs/schedule';

@Injectable()
export class ScheduledTasksService {
  @Cron(CronExpression.EVERY_HOUR)
  async handleCron() {
    // Enqueue job
    await this.emailQueue.add(Job.DailyReport, {});
  }
}
```

## Best Practices

1. **Idempotent Jobs** - Jobs should be safe to retry
2. **Small Job Data** - Keep job data small (store references)
3. **Error Handling** - Always handle errors gracefully
4. **Logging** - Log job progress and failures
5. **Monitoring** - Monitor job queues and failures
6. **Rate Limiting** - Use limiters to prevent overload
7. **Cleanup** - Remove old completed/failed jobs
8. **Testing** - Test job processors thoroughly

## Troubleshooting

### Jobs Not Processing

1. **Check worker is running**: `IS_WORKER=true`
2. **Check Redis connection**: Verify Redis is accessible
3. **Check queue name**: Ensure queue name matches
4. **Check processor registration**: Verify processor is registered

### Jobs Failing

1. **Check logs**: Review processor logs
2. **Check job data**: Verify job data format
3. **Check dependencies**: Ensure all dependencies available
4. **Check retry logic**: Review retry configuration

### Performance Issues

1. **Adjust concurrency**: Increase if CPU allows
2. **Optimize processing**: Improve job processing logic
3. **Use limiters**: Prevent overwhelming services
4. **Monitor resources**: Check CPU/memory usage

## Next Steps

- Review [Configuration](./08-CONFIGURATION.md) for queue configuration
- Check [Deployment Guide](./10-DEPLOYMENT.md) for worker deployment
- See [Monitoring Guide](./11-MONITORING.md) for job monitoring

