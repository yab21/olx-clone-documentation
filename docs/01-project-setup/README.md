# Project Setup

This guide will walk you through the initial setup of your OLX clone project, including both the frontend (Next.js) and backend (Node.js/Express) components.

## Directory Structure

Create a main project directory with separate folders for client and server:

```bash
mkdir olx-clone
cd olx-clone
```

## Frontend Setup (Next.js)

### Step 1: Initialize the Next.js project

```bash
npx create-next-app@latest client --typescript
cd client
```

When prompted, choose these options:
- TypeScript: Yes
- ESLint: Yes
- Tailwind CSS: Yes
- `src/` directory: Yes
- App Router: Yes
- Import alias: Yes (default @/*)

### Step 2: Install additional dependencies

```bash
npm install axios react-hook-form zod @hookform/resolvers next-auth react-icons react-toastify @tailwindcss/forms
```

### Step 3: Configure Tailwind CSS

The setup is handled by the Next.js initialization, but you might want to add custom theme values in `tailwind.config.js`:

```js
/** @type {import('tailwindcss').Config} */
module.exports = {
  darkMode: ["class"],
  content: [
    "./pages/**/*.{ts,tsx}",
    "./components/**/*.{ts,tsx}",
    "./app/**/*.{ts,tsx}",
    "./src/**/*.{ts,tsx}",
  ],
  theme: {
    extend: {
      colors: {
        primary: "#002f34", // OLX-like primary color
        secondary: "#23e5db", // OLX-like secondary color
      },
      container: {
        center: true,
        padding: "2rem",
        screens: {
          "2xl": "1400px",
        },
      },
    },
  },
  plugins: [require("@tailwindcss/forms")],
};
```

### Step 4: Set up environment variables

Create a `.env.local` file in the client directory:

```
NEXT_PUBLIC_API_URL=http://localhost:5000/api
NEXTAUTH_SECRET=your-nextauth-secret
NEXTAUTH_URL=http://localhost:3000
```

## Backend Setup (Node.js/Express)

### Step 1: Initialize the Node.js project

```bash
mkdir server
cd server
npm init -y
```

### Step 2: Install dependencies

```bash
npm install express mongoose dotenv cors bcrypt jsonwebtoken multer morgan validator helmet compression
npm install --save-dev typescript ts-node nodemon @types/express @types/node @types/cors @types/bcrypt @types/jsonwebtoken @types/multer @types/morgan
```

### Step 3: Initialize TypeScript

```bash
npx tsc --init
```

Edit `tsconfig.json` to include:

```json
{
  "compilerOptions": {
    "target": "es6",
    "module": "commonjs",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules"]
}
```

### Step 4: Create server directory structure

```bash
mkdir -p src/controllers src/models src/routes src/middlewares src/config src/utils src/uploads
```

### Step 5: Create environment variables

Create a `.env` file in the server directory:

```
NODE_ENV=development
PORT=5000
MONGO_URI=mongodb://localhost:27017/olx-clone
JWT_SECRET=your-jwt-secret-key
JWT_EXPIRE=30d
```

### Step 6: Create the initial server file

Create a file `src/server.ts`:

```typescript
import express, { Express } from 'express';
import dotenv from 'dotenv';
import cors from 'cors';
import helmet from 'helmet';
import morgan from 'morgan';
import path from 'path';
import compression from 'compression';
import { connectDB } from './config/db';

// Load environment variables
dotenv.config();

// Initialize express app
const app: Express = express();
const PORT = process.env.PORT || 5000;

// Connect to MongoDB
connectDB();

// Middleware
app.use(cors());
app.use(express.json());
app.use(helmet());
app.use(compression());
app.use(morgan('dev'));

// Static files directory for uploaded images
app.use('/uploads', express.static(path.join(__dirname, '../uploads')));

// Basic route
app.get('/', (req, res) => {
  res.send('API running');
});

// Import routes here
// app.use('/api/users', require('./routes/userRoutes'));
// app.use('/api/listings', require('./routes/listingRoutes'));
// app.use('/api/messages', require('./routes/messageRoutes'));

// Error handler middleware (to be created)
// app.use(errorHandler);

app.listen(PORT, () => {
  console.log(`Server running in ${process.env.NODE_ENV} mode on port ${PORT}`);
});
```

### Step 7: Create database connection

Create a file `src/config/db.ts`:

```typescript
import mongoose from 'mongoose';

export const connectDB = async (): Promise<void> => {
  try {
    if (!process.env.MONGO_URI) {
      throw new Error('MONGO_URI is not defined in environment variables');
    }
    
    const conn = await mongoose.connect(process.env.MONGO_URI);
    
    console.log(`MongoDB Connected: ${conn.connection.host}`);
  } catch (error) {
    if (error instanceof Error) {
      console.error(`Error: ${error.message}`);
    } else {
      console.error('Unknown error occurred');
    }
    process.exit(1);
  }
};
```

### Step 8: Update package.json for server

Edit the server's package.json to include these scripts:

```json
"scripts": {
  "start": "node dist/server.js",
  "dev": "nodemon src/server.ts",
  "build": "tsc -p ."
}
```

## Git Setup

Initialize Git in the root directory:

```bash
cd ..  # Return to root directory
git init
```

Create a `.gitignore` file:

```
# dependencies
node_modules
/.pnp
.pnp.js

# testing
/coverage

# next.js
/.next/
/out/

# production
/build
/dist

# misc
.DS_Store
*.pem

# debug
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# local env files
.env
.env.local
.env.development.local
.env.test.local
.env.production.local

# vercel
.vercel

# typescript
*.tsbuildinfo
next-env.d.ts

# uploads
/server/uploads/*
!/server/uploads/.gitkeep
```

Create empty `.gitkeep` file to maintain uploads directory:

```bash
mkdir -p server/uploads
touch server/uploads/.gitkeep
```

## Starting the Development Servers

### Frontend

```bash
cd client
npm run dev
```

This will start the Next.js development server at http://localhost:3000.

### Backend

```bash
cd server
npm run dev
```

This will start the Express server at http://localhost:5000.

## Next Steps

Once you have completed the initial setup, you can proceed to the [Database Design](../02-database-design/README.md) section to set up your MongoDB schemas.