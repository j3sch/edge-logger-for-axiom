# Edge Logger for Axiom

OpenTelemetry aware logger for Edge Runtimes.

Fully tested with:

-   Cloudflare Workers
-   Cloudflare Pages Functions
-   Vercel Edge Functions

## Usage

1. Install dependencies:

    ```bash
    npm i -D @cloudflare/workers-types
    npm i @opentelemetry/api
    ```

2. Create Edge Logger file:

    ```ts
    //edge-logger.ts
    import type { ExecutionContext } from '@cloudflare/workers-types'
    import { trace } from '@opentelemetry/api'

    export type Log = {
        message: string
        error?: string
        requestId: string
        level: string
        traceId: string | undefined
        [key: string]: unknown
    }

    export type LoggerArgs = {
        ctx: ExecutionContext
        apiKey: string
        dataset?: string
        service?: string
        apiURL?: string
        flushAfterMs?: number
        flushAfterLogs?: number
        requestId?: string | null
        isLocalDev?: boolean
    }

    type LogLevel = 'info' | 'warning' | 'error' | 'debug'

    export class EdgeLogger {
        private readonly ctx: ExecutionContext
        private readonly apiKey: string
        private readonly dataset: string
        private readonly service: string | undefined
        private readonly logs: Log[] = []
        private readonly requestId: string
        // flushTimeout is a timeout set by setTimeout() to flush the logs after a certain amount of time
        private flushTimeout: NodeJS.Timeout | null = null
        private flushPromise: Promise<void> | null = null
        private flushAfterMs: number
        private flushAfterLogs: number
        private apiURL: string
        private isLocalDev: boolean

        constructor(args: LoggerArgs) {
            this.ctx = args.ctx
            this.apiKey = args.apiKey
            this.dataset = args.dataset ?? 'edge-logger'
            this.service = args.service
            this.flushAfterMs = args.flushAfterMs ?? 10000
            this.flushAfterLogs = args.flushAfterLogs ?? 100
            this.apiURL = args.apiURL ?? 'https://api.axiom.co/v1'
            if (args.requestId) {
                this.requestId = args.requestId
            } else {
                this.requestId = crypto.randomUUID()
            }
            this.isLocalDev = args.isLocalDev ?? false
        }

        private async _log(message: string, level: LogLevel = 'info', data?: Record<string, unknown>) {
            const span = trace.getActiveSpan()
            const traceId = span?.spanContext().traceId

            if (this.isLocalDev) {
                const colors = {
                    info: '\x1b[32m',
                    warning: '\x1b[33m',
                    error: '\x1b[31m',
                    debug: '\x1b[35m',
                }

                const grey = '\x1b[90m'
                const white = '\x1b[0m'
                console.log(`${colors[level]}${level}${grey} - ${this.requestId} - ${white}${message}`)
                if (data) {
                    console.log(`${grey} ${JSON.stringify(data, null, 2)}`)
                }
                return
            }

            const log: Log = {
                message,
                level: (data?.level as string) || level,
                traceId,
                _time: new Date().toISOString(),
                service: this.service,
                requestId: this.requestId,
                ...data,
            }
            /**
             * If no API key or context, we can't send logs so log them to sdtout
             */
            if (!this.apiKey || !this.ctx) {
                console.log(JSON.stringify(log))
            }
            this.logs.push(log)

            if (this.logs.length >= this.flushAfterLogs) {
                // Reset scheduled if there is one
                if (this.flushTimeout) {
                    this.scheduleFlush(this.flushAfterMs, true)
                }
                this.ctx.waitUntil(this.flush({ skipIfInProgress: true }))
            } else {
                // Always schedule a flush (if there isn't one already)
                this.scheduleFlush(this.flushAfterMs)
            }
        }

        /** Flush after X ms if there's not already
         * a flush scheduled
         * @param reset If true, cancel the current flush timeout
         */
        scheduleFlush(timeout: number, reset = false) {
            if (reset && this.flushTimeout) {
                clearTimeout(this.flushTimeout)
                this.flushTimeout = null
            }

            if (!this.flushTimeout && !this.flushPromise) {
                this.flushTimeout = setTimeout(() => {
                    const doFlush = async () => {
                        this.flush({ skipIfInProgress: true })
                        this.flushTimeout = null
                    }
                    this.ctx.waitUntil(doFlush())
                }, timeout)
            }
        }

        async flush({ skipIfInProgress = false }: { skipIfInProgress?: boolean } = {}) {
            if (skipIfInProgress && this.flushPromise) return
            const doFlush = async () => {
                if (this.logs.length === 0) return // Nothing to do

                const logsCount = this.logs.length
                const logsBody = JSON.stringify(this.logs)

                try {
                    const res = await fetch(`${this.apiURL}/datasets/${this.dataset}/ingest`, {
                        method: 'POST',
                        headers: {
                            'Content-Type': 'application/json',
                            Authorization: `Bearer ${this.apiKey}`,
                        },
                        body: logsBody,
                    })
                    if (res.ok) {
                        // Remove the logs we sent
                        this.logs.splice(0, logsCount)
                        await res.arrayBuffer() // Read the body to completion
                    } else {
                        console.log(`Axiom failed to ingest logs: ${res.status} ${res.statusText} ${await res.text()}`)
                    }
                } catch (err) {
                    console.error(`Axiom failed to ingest logs: ${err}`)
                }
            }

            // Make sure the last one is done before starting a flush
            await this.flushPromise

            this.flushPromise = doFlush()
            await this.flushPromise
            this.flushPromise = null
        }

        log(msg: string, data?: Record<string, unknown>) {
            this._log(msg, 'info', data)
        }

        info(msg: string, data?: Record<string, unknown>) {
            this._log(msg, 'info', data)
        }

        warn(msg: string, data?: Record<string, unknown>) {
            this._log(msg, 'warning', data)
        }

        error(msg: string | Error | unknown, data?: Record<string, unknown>) {
            let m = ''
            if (msg instanceof Error) {
                m = msg.message + (msg.stack ? `: ${msg.stack}` : '')
            } else if (typeof msg === 'string') {
                m = msg
            } else {
                m = JSON.stringify(msg)
            }
            this._log(m, 'error', data)
        }

        debug(msg: string, data?: Record<string, unknown>) {
            this._log(msg, 'debug', data)
        }
    }
    ```

3. Use the EdgeLogger:

    ```ts
    import { EdgeLogger } from './edge-logger'

    export interface Env {
        AXIOM_TOKEN: string
        AXIOM_DATASET: string
    }

    export default {
        async fetch(req: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
            const logger = new EdgeLogger({
                ctx,
                apiKey: env.AXIOM_TOKEN,
                dataset: env.AXIOM_DATASET,
                service: 'my-service',
                requestId: crypto.randomUUID(),
            })

            // Use the EdgeLogger to log
            logger.info('Hello, World!', { foo: 'bar' })

            // Make sure to add this to flush the logs to Axiom before the return statement
            ctx.waitUntil(logger.flush())
            return new Response('Request Completed')
        },
    }
    ```

> Note the `ctx.waitUntil(logger.flush());` before the return statement of the function.

## Supported Methods

```typescript
logger.info('This is an informational message', { payload: { foo: 'bar' } })
logger.warn('This is a warning message', { payload: { foo: 'bar' } })
logger.error('This is an error message', { payload: { foo: 'bar' } })
```

## Local Development

To get logs appropriately formatted in your local development environment using wrangler, add the `IS_LOCAL_DEV` environment variable to your `.dev.var` file.

```.env
IS_LOCAL_MODE=1
```

And use it when configuring the `EdgeLogger`.

```typescript
export default {
    async fetch(req: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
        const logger = new EdgeLogger({
            ctx,
            apiKey: env.BASELIME_KEY,
            isLocalDev: env.IS_LOCAL_DEV,
        })

        logger.info('Pretty Formatted Logs', { foo: 'bar' })

        ctx.waitUntil(logger.flush())
    },
}
```

Local development mode is disabled by default to improve performance.

## Configuration

The BaselimeLogger class takes the following configuration options

| Property         | Type                 | Description                             |
| ---------------- | -------------------- | --------------------------------------- |
| `ctx`            | `ExecutionContext`   | Execution context                       |
| `apiKey`         | `string`             | API key for authentication              |
| `dataset`        | `string` (optional)  | Dataset name                            |
| `service`        | `string` (optional)  | Service name                            |
| `namespace`      | `string` (optional)  | Namespace                               |
| `flushAfterMs`   | `number` (optional)  | Wait time to flush the logs to Baselime |
| `flushAfterLogs` | `number` (optional)  | Threshold number of logs to flush       |
| `requestId`      | `string` (optional)  | Request ID                              |
| `isLocalDev`     | `boolean` (optional) | Indicates if it's for local development |

## Contributor Alumni

The Edge Logger was created for the [Baselime community](https://join.slack.com/t/baselimecommunity/shared_invite/zt-25rig7ul2-m06V0DdYWpFHu~qH3F7DhQ) and generously offered to Baselime by [Jacob Hands](https://github.com/jahands).

### Contributing

If you would like to contribute to the development of this library, please submit a pull request on GitHub.

### License

This library is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.
