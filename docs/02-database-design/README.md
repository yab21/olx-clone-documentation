# Database Design

This section details the MongoDB schema design for the OLX clone project. We'll use Mongoose as our ODM (Object Data Modeling) library to create schemas and models.

## Overview

The database for our OLX clone will consist of the following main collections:
- Users
- Listings
- Categories
- Messages

## User Schema

The User schema handles user authentication and profile information.

Create a file `src/models/userModel.ts`:

```typescript
import mongoose from 'mongoose';
import bcrypt from 'bcrypt';
import validator from 'validator';

export interface IUser extends mongoose.Document {
  name: string;
  email: string;
  password: string;
  phone?: string;
  location?: string;
  avatar?: string;
  createdAt: Date;
  updatedAt: Date;
  matchPassword(enteredPassword: string): Promise<boolean>;
}

const userSchema = new mongoose.Schema<IUser>(
  {
    name: {
      type: String,
      required: [true, 'Please add a name'],
      trim: true,
      maxlength: [50, 'Name cannot be more than 50 characters'],
    },
    email: {
      type: String,
      required: [true, 'Please add an email'],
      unique: true,
      trim: true,
      lowercase: true,
      validate: [validator.isEmail, 'Please provide a valid email'],
    },
    password: {
      type: String,
      required: [true, 'Please add a password'],
      minlength: [6, 'Password must be at least 6 characters'],
      select: false, // Don't return password in queries
    },
    phone: {
      type: String,
      trim: true,
      validate: {
        validator: function(v: string) {
          return validator.isMobilePhone(v);
        },
        message: 'Please provide a valid phone number',
      },
    },
    location: {
      type: String,
      trim: true,
    },
    avatar: {
      type: String,
      default: 'default-avatar.jpg',
    },
  },
  {
    timestamps: true,
  }
);

// Encrypt password using bcrypt
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) {
    next();
  }

  const salt = await bcrypt.genSalt(10);
  this.password = await bcrypt.hash(this.password, salt);
});

// Match user entered password to hashed password in database
userSchema.methods.matchPassword = async function(enteredPassword: string): Promise<boolean> {
  return await bcrypt.compare(enteredPassword, this.password);
};

const User = mongoose.model<IUser>('User', userSchema);

export default User;
```

## Category Schema

The Category schema defines product/service categories and subcategories.

Create a file `src/models/categoryModel.ts`:

```typescript
import mongoose from 'mongoose';

export interface ICategory extends mongoose.Document {
  name: string;
  slug: string;
  icon?: string;
  parent?: mongoose.Types.ObjectId | null;
  createdAt: Date;
  updatedAt: Date;
}

const categorySchema = new mongoose.Schema<ICategory>(
  {
    name: {
      type: String,
      required: [true, 'Please add a category name'],
      trim: true,
      maxlength: [50, 'Name cannot be more than 50 characters'],
    },
    slug: {
      type: String,
      required: true,
      unique: true,
      trim: true,
      lowercase: true,
    },
    icon: {
      type: String,
    },
    parent: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'Category',
      default: null,
    },
  },
  {
    timestamps: true,
    toJSON: { virtuals: true },
    toObject: { virtuals: true },
  }
);

// Virtual for subcategories
categorySchema.virtual('subcategories', {
  ref: 'Category',
  localField: '_id',
  foreignField: 'parent',
});

const Category = mongoose.model<ICategory>('Category', categorySchema);

export default Category;
```

## Listing Schema

The Listing schema is the core of our marketplace, representing items for sale.

Create a file `src/models/listingModel.ts`:

```typescript
import mongoose from 'mongoose';

export interface IListing extends mongoose.Document {
  title: string;
  description: string;
  price: number;
  category: mongoose.Types.ObjectId;
  images: string[];
  location: string;
  seller: mongoose.Types.ObjectId;
  status: 'active' | 'sold' | 'expired';
  featured: boolean;
  views: number;
  createdAt: Date;
  updatedAt: Date;
}

const listingSchema = new mongoose.Schema<IListing>(
  {
    title: {
      type: String,
      required: [true, 'Please add a title'],
      trim: true,
      maxlength: [100, 'Title cannot be more than 100 characters'],
    },
    description: {
      type: String,
      required: [true, 'Please add a description'],
      trim: true,
      maxlength: [2000, 'Description cannot be more than 2000 characters'],
    },
    price: {
      type: Number,
      required: [true, 'Please add a price'],
      min: [0, 'Price must be positive'],
    },
    category: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'Category',
      required: [true, 'Please select a category'],
    },
    images: {
      type: [String],
      required: [true, 'Please add at least one image'],
      validate: {
        validator: function(v: string[]) {
          return v.length > 0 && v.length <= 10;
        },
        message: 'Please add between 1 and 10 images',
      },
    },
    location: {
      type: String,
      required: [true, 'Please add a location'],
      trim: true,
    },
    seller: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'User',
      required: true,
    },
    status: {
      type: String,
      enum: ['active', 'sold', 'expired'],
      default: 'active',
    },
    featured: {
      type: Boolean,
      default: false,
    },
    views: {
      type: Number,
      default: 0,
    },
  },
  {
    timestamps: true,
  }
);

// Create indexes for search operations
listingSchema.index({ title: 'text', description: 'text' });
listingSchema.index({ location: 1 });
listingSchema.index({ category: 1 });
listingSchema.index({ seller: 1 });
listingSchema.index({ status: 1 });
listingSchema.index({ createdAt: -1 });

const Listing = mongoose.model<IListing>('Listing', listingSchema);

export default Listing;
```

## Message Schema

The Message schema handles communication between users.

Create a file `src/models/messageModel.ts`:

```typescript
import mongoose from 'mongoose';

export interface IMessage extends mongoose.Document {
  listing: mongoose.Types.ObjectId;
  sender: mongoose.Types.ObjectId;
  receiver: mongoose.Types.ObjectId;
  content: string;
  read: boolean;
  createdAt: Date;
  updatedAt: Date;
}

const messageSchema = new mongoose.Schema<IMessage>(
  {
    listing: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'Listing',
      required: true,
    },
    sender: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'User',
      required: true,
    },
    receiver: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'User',
      required: true,
    },
    content: {
      type: String,
      required: [true, 'Message content is required'],
      trim: true,
      maxlength: [1000, 'Message cannot be more than 1000 characters'],
    },
    read: {
      type: Boolean,
      default: false,
    },
  },
  {
    timestamps: true,
  }
);

// Create indexes for better query performance
messageSchema.index({ sender: 1, receiver: 1 });
messageSchema.index({ listing: 1 });
messageSchema.index({ read: 1 });

const Message = mongoose.model<IMessage>('Message', messageSchema);

export default Message;
```

## Favorite/Saved Listing Schema

To allow users to save their favorite listings, we'll create another schema:

Create a file `src/models/favoriteModel.ts`:

```typescript
import mongoose from 'mongoose';

export interface IFavorite extends mongoose.Document {
  user: mongoose.Types.ObjectId;
  listing: mongoose.Types.ObjectId;
  createdAt: Date;
}

const favoriteSchema = new mongoose.Schema<IFavorite>(
  {
    user: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'User',
      required: true,
    },
    listing: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'Listing',
      required: true,
    },
  },
  {
    timestamps: true,
  }
);

// Ensure a user can only favorite a listing once
favoriteSchema.index({ user: 1, listing: 1 }, { unique: true });

const Favorite = mongoose.model<IFavorite>('Favorite', favoriteSchema);

export default Favorite;
```

## Import Models in Server

Update your `src/server.ts` to import all models at startup:

```typescript
// Import models (place this after MongoDB connection)
import './models/userModel';
import './models/categoryModel';
import './models/listingModel';
import './models/messageModel';
import './models/favoriteModel';
```

## Database Relationships

Here's a summary of the relationships between our collections:

1. **User to Listings**: One-to-Many (One user can have many listings)
2. **Category to Listings**: One-to-Many (One category can have many listings)
3. **Category to Subcategories**: One-to-Many (One category can have many subcategories)
4. **User to Messages**: One-to-Many (One user can send/receive many messages)
5. **Listing to Messages**: One-to-Many (One listing can have many inquiries/messages)
6. **User to Favorites**: One-to-Many (One user can have many favorite listings)

## Database Seeding

For development, you may want to create seed data to test your application. Create a file `src/config/seedData.ts`:

```typescript
import mongoose from 'mongoose';
import dotenv from 'dotenv';
import User from '../models/userModel';
import Category from '../models/categoryModel';
import Listing from '../models/listingModel';
import { connectDB } from './db';

dotenv.config();

// Sample categories
const categories = [
  {
    name: 'Electronics',
    slug: 'electronics',
    icon: 'laptop',
  },
  {
    name: 'Vehicles',
    slug: 'vehicles',
    icon: 'car',
  },
  {
    name: 'Property',
    slug: 'property',
    icon: 'home',
  },
  {
    name: 'Furniture',
    slug: 'furniture',
    icon: 'chair',
  },
  {
    name: 'Fashion',
    slug: 'fashion',
    icon: 'shirt',
  },
];

// Sample subcategories
const subcategories = [
  {
    name: 'Mobile Phones',
    slug: 'mobile-phones',
    icon: 'smartphone',
    parent: 'electronics',
  },
  {
    name: 'Laptops',
    slug: 'laptops',
    icon: 'laptop',
    parent: 'electronics',
  },
  {
    name: 'Cars',
    slug: 'cars',
    icon: 'car',
    parent: 'vehicles',
  },
  {
    name: 'Apartments for Rent',
    slug: 'apartments-for-rent',
    icon: 'apartment',
    parent: 'property',
  },
];

// Import seed data
const importData = async () => {
  try {
    await connectDB();
    
    // Clear existing data
    await User.deleteMany({});
    await Category.deleteMany({});
    await Listing.deleteMany({});

    // Create admin user
    const adminUser = await User.create({
      name: 'Admin User',
      email: 'admin@example.com',
      password: 'password123',
      phone: '1234567890',
      location: 'New York',
    });

    console.log('Admin user created');

    // Create categories
    const createdCategories = await Category.insertMany(categories);
    console.log('Categories created');

    // Create subcategories
    for (const subcat of subcategories) {
      const parent = createdCategories.find(c => c.slug === subcat.parent);
      if (parent) {
        await Category.create({
          name: subcat.name,
          slug: subcat.slug,
          icon: subcat.icon,
          parent: parent._id,
        });
      }
    }
    console.log('Subcategories created');

    console.log('Data imported successfully');
    process.exit(0);
  } catch (error) {
    console.error(`Error: ${error}`);
    process.exit(1);
  }
};

// Delete all data
const destroyData = async () => {
  try {
    await connectDB();
    await User.deleteMany({});
    await Category.deleteMany({});
    await Listing.deleteMany({});

    console.log('Data destroyed successfully');
    process.exit(0);
  } catch (error) {
    console.error(`Error: ${error}`);
    process.exit(1);
  }
};

// Run seed script
if (process.argv[2] === '-d') {
  destroyData();
} else {
  importData();
}
```

Add these scripts to your server's `package.json`:

```json
"scripts": {
  "data:import": "ts-node src/config/seedData.ts",
  "data:destroy": "ts-node src/config/seedData.ts -d"
}
```

To seed your database, run:

```bash
npm run data:import
```

## Next Steps

Now that you have designed and implemented your database schemas, proceed to the [Backend Development](../03-backend-development/README.md) section to build the API endpoints that will interact with these models.