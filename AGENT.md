# MSolo overview
MSolo is a MapleStory V83 private server. This repository is the server.
Refer to `\docs\plan.md` for the project plan and context.
Refer to `\docs\quests.md` for the quest system architecture, flows, and roadmap.

## Entrypoints and scripts
- `\src\main\java\net\server\Server.java` - server start entrypoint via the `main` method.

## Architecture
- `MySQL 9.7.1`: DB used for server data.
- `Java 21`: Used for main server implementation.
- `JavaScript`: Used for various script executions.
- `Maven`: Project build system.
