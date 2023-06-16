To create a Discord bot using
<span style="color:#ADD8E6">**Discord.js v14**</span>,
<span style= "color: #C3A774">**MongoDB**</span>,
and <span style="color:#ADD8E6">TypeScript</span> in a Visual Studio Code project, follow these steps:

# Install Node.js

Make sure you have **[Node.js](https://nodejs.org/)** installed on your system.

- _<span style="color:#ADD8E6">Discord.js v14</span> requires <span style="color:#98FB98">Node.js v16.6.0</span> or higher._

# Create a new project folder

Create a new folder for your project and open it in Visual Studio Code.

# Initialize the project:

Open the terminal in Visual Studio Code and run the following command to create a package.json file:

```powershell
npm init -y
```

# Installing NPM Packages

In the terminal, run the following command to install all the required NPM Packages for this project:

```powershell
npm install discord.js @discordjs/builders typescript --save-dev ts-node @types/node dotenv fs path mongodb
```

# Structure

Before we talk about creating files, here's an ASCII art representation of the project directory structure:  
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

# tsconfig.json

Create a new file in your project folder called `tsconfig.json` and add the following configuration:

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

Inside the <span style= "color: #C3A774">package.json</span> file, add/replace the default section of

```json
"scripts": {  }
```

with the following:

```json
"scripts": {
  "start": "ts-node src/bot.ts",
  "build": "tsc",
  "dev": "tsc-watch --onSuccess \"node ./dist/bot.js\""
}
```

# Set up your bot token

Go to the **[Discord Developer Portal](https://discord.com/developers/applications)** and create a new application. Navigate to the "Bot" tab and click "Add Bot". Copy the bot token and store it securely. You should **<u> never </u>** share your bot token with anyone.

## Create a .env file:

Create a new file in your project folder called `.env` and add the following line:

```env
DISCORD_BOT_TOKEN=your_bot_token
```

Replace `your_bot_token` with the token you copied from the Discord Developer Portal.

# Source Folder

Create a `src` folder and `bot.ts` file:  
Create a new folder called `src` in your project folder. Inside the `src` folder, create a new file called `bot.ts`. This file will your bot's code. We will come back to this at the end as we must setup everything else.

## Setting Up MongoDB

Create a `config.ts` file in the `src/config` folder to store your MongoDB connection string:

### src/config/config.ts

```ts
export const config = {
  mongoURI: "your_mongodb_connection_string",
};
```

## Creating our utils folder

Create the `utils` folder, `registry.ts` and `permission.ts` file: Create a new folder called `utils` inside the `src` folder.

Inside the `utils` folder, create a new file called `registry.ts`. This file will contain the functions to register commands and events:

### src/utils/registry.ts

```ts
import { Client, Collection } from "discord.js";
import { readdirSync, statSync } from "fs";
import { MongoClient } from "mongodb";
import { join } from "path";

declare module "discord.js" {
  interface Client {
    commands: Collection<string, any>;
    events: Collection<string, any>;
    mongo: MongoClient;
  } /* 
  Here we are declaring properties on the Client interface. 
  - In JavaScript, you can add any property you want without declaring anything. 
  - In TypeScript, you must declare these as you will be given an error otherwise. 
  - These can be accessed anywhere in our project.
  */
}

export async function registerCommands(client: Client, dir: string) {
  client.commands = new Collection(); // Initialize the commands property
  const files = readdirSync(join(__dirname, dir));
  for (const file of files) {
    const filePath = join(__dirname, dir, file);
    const stat = statSync(filePath); // Use statSync instead of readdirSync - GPT-4 Error (GPT-4 corrected)
    if (stat.isDirectory()) {
      registerCommands(client, join(dir, file));
    } else {
      if (file.endsWith(".ts")) {
        const { command } = await import(filePath);
        client.commands.set(command.data.name, command);
      }
    }
  }
}

export async function registerEvents(client: Client, dir: string) {
  const files = readdirSync(join(__dirname, dir));
  for (const file of files) {
    const filePath = join(__dirname, dir, file);
    const stat = statSync(filePath); // Use statSync instead of readdirSync - GPT-4 Error (GPT-4 corrected)
    if (stat.isDirectory()) {
      registerEvents(client, join(dir, file));
    } else {
      if (file.endsWith(".ts")) {
        const { event } = await import(filePath);
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

Inside the `utils` folder, create a new file called `permission.ts`. You can create a utility function to check the user's permission level in the utils folder. This function will query the database and return the user's permission level. If the user is not in the database, it will return the default permission level (1).

### src/utils/permission.ts

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
  /*
   A question mark is used after a condition followed by an expression and a colon. 
   This is called the ternary operator which serves to offer two solutions: 
   - The question mark is if the condition is truthy
   - The colon is if the condition is falsy
   */

  return userPermissionLevel >= requiredPermissionLevel;
};
```

## Create the events and commands folders:

Inside the `src` folder, create two new folders called `events` and `commands`. Inside the `commands` folder, create subfolders for different categories of commands (e.g., util, misc, moderation, meme, etc.).

### Create an example event file:

Inside the events folder, create a new file called `ready.ts`:

#### src/events/ready.ts

```ts
import { Client } from "discord.js";

export const event = {
  name: "ready",
  once: true,
  execute(client: Client) {
    console.log(`Logged in as ${client.user?.tag}!`); // client.user can return null, so the question mark; aka optional, will stop typescript from giving a warning/error.
  },
};
```

### Create an example command file:

Inside the commands/moderation folder, create a new file called `ping.ts:`

#### src/commands/moderation/ping.ts

```ts
import { SlashCommandBuilder } from "@discordjs/builders";
import { CommandInteraction } from "discord.js";
import { MongoClient } from "mongodb";
import { checkPermission } from "../../utils/permission"; // importing missing function (GPT-4 Corrected)

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

## src/bot.ts

Now the moment you've been waiting for, here is the following code for <span style="color:#ADD8E6">bot.ts</span>

```ts
import { Client, Collection, IntentsBitField } from "discord.js";
import { MongoClient } from "mongodb";
import dotenv from "dotenv";
import { config } from "./config/config";
import { registerCommands, registerEvents } from "./utils/registry";

dotenv.config();

(async () => {
  const client = new Client({
    intents: [
      IntentsBitField.Flags.Guilds,
      IntentsBitField.Flags.GuildMessages,
    ],
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

# Author's Note: GPT-4 Corrections

As you can see above, **GPT-4** did make some errors (3), however it did correct it's own errors too after submitting it's own code.

The conversation is hyperlinked **[here](https://chat.forefront.ai/share/faarw5gfcn6d8f7l)**, on a website that allows you to use GPT-4, or you can click the following link: **https://chat.forefront.ai/share/faarw5gfcn6d8f7l**,

<a href="https://ibb.co/Yp871Kh"><img src="https://i.ibb.co/yW5nZMq/image.png" alt="A conversation of the code error (1/2)" border="0"></a>
<a href="https://ibb.co/nb7HjCQ"><img src="https://i.ibb.co/BtyXBNT/image-1.png" alt="A conversation of the code error (2/2)" border="0"></a>
