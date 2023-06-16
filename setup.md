<span style="color:#D2B48C">**your-bot/**</span>  
├── <span style="color:#D2B48C">**src/**</span>  
│ ├── <span style="color:#D2B48C">**commands/**</span>  
│ │ ├── <span style="color:#D2B48C">**moderation/**</span>  
│ │ │ └── <span style="color:#ADD8E6">ban.ts</span>  
│ │ └── <span style="color:#D2B48C">**util/**</span>  
│ │ └── <span style="color:#ADD8E6">ping.ts</span>  
│ ├── <span style="color:#D2B48C">**config/**</span>  
│ │ └── <span style="color:#ADD8E6">config.ts</span>  
│ ├── <span style="color:#D2B48C">**events/**</span>  
│ │ ├── <span style="color:#ADD8E6">messageCreate.ts</span>  
│ │ └── <span style="color:#ADD8E6">ready.ts</span>  
│ ├── <span style="color:#D2B48C">**utils/**</span>  
│ │ ├── <span style="color:#ADD8E6">permissions.ts</span>  
│ │ └── <span style="color:#ADD8E6">registry.ts</span>  
│ └── <span style="color:#ADD8E6">bot.ts</span>  
├── <span style= "color: #C3A774">package.json</span>  
└── <span style= "color: #C3A774">tsconfig.json</span>

# src/config/config.ts

```ts
export const config = {
  mongoURI: "your_mongodb_connection_string",
};
```

# src/commands/moderation/ban.ts

```ts
import { CommandInteraction, Client } from "discord.js";
import { SlashCommandBuilder } from "@discordjs/builders";

export const command = {
  data: new SlashCommandBuilder()
    .setName("ban")
    .setDescription("Ban a user")
    .addUserOption((option) =>
      option.setName("user").setDescription("The user to ban").setRequired(true)
    )
    .addStringOption((option) =>
      option
        .setName("reason")
        .setDescription("The reason for the ban")
        .setRequired(false)
    ),
  async execute(interaction: CommandInteraction, client: Client) {
    // Check permissions and execute the command
  },
};
```

# src/commands/util/ping.ts

```ts
import { SlashCommandBuilder } from "@discordjs/builders";
import { CommandInteraction } from "discord.js";
import { MongoClient } from "mongodb";

export const data = new SlashCommandBuilder()
  .setName("ping")
  .setDescription("Replies with Pong!");

export const execute = async (
  interaction: CommandInteraction,
  mongoClient: MongoClient
) => {
  const hasPermission = await checkPermission(interaction, mongoClient, 2);

  if (hasPermission) {
    await interaction.reply("Pong!");
  } else {
    await interaction.reply({
      content: "You do not have permission to use this command.",
      ephemeral: true,
    });
  }
};
```

# src/events/messageCreate.ts

```ts
import { Message } from "discord.js";
import { MongoClient } from "mongodb";

export const name = "messageCreate";

export const execute = async (message: Message, mongoClient: MongoClient) => {
  // Your message handling code here
};
```

# src/events/ready.ts

```ts
import { Client } from "discord.js";
import { MongoClient } from "mongodb";

export const name = "ready";
export const once = true;

export const execute = (client: Client, mongoClient: MongoClient) => {
  console.log(`Logged in as ${client.user?.tag}!`);
};
```

# src/utils/permissions.ts

```ts
import { CommandInteraction } from "discord.js";
import { MongoClient } from "mongodb";

export const checkPermission = async (
  interaction: CommandInteraction,
  mongoClient: MongoClient,
  requiredPermissionLevel: number
): Promise<boolean> => {
  const userId = interaction.user.id;
  const collection = mongoClient
    .db("your_database_name")
    .collection("permissions");

  const userPermission = await collection.findOne({ userId });

  const userPermissionLevel = userPermission ? userPermission.level : 1;

  return userPermissionLevel >= requiredPermissionLevel;
};
```

# src/utils/registry.ts

```ts
import { Client } from "discord.js";
import { readdirSync } from "fs";
import { join } from "path";

export async function registerCommands(client: Client, dir: string) {
  const files = readdirSync(join(__dirname, dir));
  for (const file of files) {
    const stat = readdirSync(join(__dirname, dir, file)).isDirectory();
    if (stat) {
      registerCommands(client, join(dir, file));
    } else {
      if (file.endsWith(".ts")) {
        const { command } = await import(join(__dirname, dir, file));
        client.commands.set(command.data.name, command);
      }
    }
  }
}

export async function registerEvents(client: Client, dir: string) {
  const files = readdirSync(join(__dirname, dir));
  for (const file of files) {
    const stat = readdirSync(join(__dirname, dir, file)).isDirectory();
    if (stat) {
      registerEvents(client, join(dir, file));
    } else {
      if (file.endsWith(".ts")) {
        const { event } = await import(join(__dirname, dir, file));
        if (event.once) {
          client.once(event.name, (...args) => event.execute(...args, client));
        } else {
          client.on(event.name, (...args) => event.execute(...args, client));
        }
      }
    }
  }
}
```

# src/bot.ts

```ts
import { Client, Collection, Intents } from "discord.js";
import { MongoClient } from "mongodb";
import dotenv from "dotenv";
import { config } from "./config";
import { registerCommands, registerEvents } from "./utils/registry";

dotenv.config();

(async () => {
  const client = new Client({
    intents: [Intents.FLAGS.Guilds, Intents.FLAGS.GuildMessages],
  });
  client.commands = new Collection();
  client.events = new Collection();
  client.mongo = new MongoClient(config.mongoURI);

  await client.mongo.connect();
  console.log("Connected to MongoDB");

  await registerCommands(client, "../commands");
  await registerEvents(client, "../events");

  client.login(process.env.DISCORD_BOT_TOKEN);
})();
```

# tsconfig.json
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "outDir": "./dist",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src/**/*.ts"],
  "exclude": ["node_modules"]
}
```

# package.json
```json
"scripts": {
  "start": "ts-node src/bot.ts",
  "build": "tsc",
  "dev": "tsc-watch --onSuccess \"node ./dist/bot.js\""
}
```
# .env
```env
DISCORD_BOT_TOKEN=your_bot_token
```

# shell
```powershell
npm init -y
npm install discord.js typescript --save-dev ts-node @types/node dotenv fs path @types/mongodb mongodb @discordjs/builders
```
