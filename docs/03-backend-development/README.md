# Backend Development

This section covers the implementation of the REST API for the OLX clone using Node.js, Express, and TypeScript.

## Overview

We'll build the following API endpoints:
- Authentication routes
- User routes
- Listing routes
- Category routes
- Message routes
- Search routes

## Utility Functions

First, let's create some utility functions that will be used throughout the application.

Create a file `src/utils/errorResponse.ts`:

```typescript
class ErrorResponse extends Error {
  statusCode: number;

  constructor(message: string, statusCode: number) {
    super(message);
    this.statusCode = statusCode;
  }
}

export default ErrorResponse;
```

Create a JWT utility at `src/utils/jwtUtils.ts`:

```typescript
import jwt from 'jsonwebtoken';
import { IUser } from '../models/userModel';

// Generate JWT token
export const generateToken = (user: IUser): string => {
  return jwt.sign(
    { id: user._id },
    process.env.JWT_SECRET || 'default_secret',
    {
      expiresIn: process.env.JWT_EXPIRE || '30d',
    }
  );
};

// Verify JWT token
export const verifyToken = (token: string): any => {
  return jwt.verify(token, process.env.JWT_SECRET || 'default_secret');
};
```

## Middleware

Let's create middleware for error handling, authentication, and image uploads.

Create a file `src/middlewares/errorMiddleware.ts`:

```typescript
import { Request, Response, NextFunction } from 'express';
import ErrorResponse from '../utils/errorResponse';

const errorHandler = (
  err: any,
  req: Request,
  res: Response,
  next: NextFunction
) => {
  let error = { ...err };
  error.message = err.message;

  // Log error for dev
  console.error(err);

  // Mongoose bad ObjectId
  if (err.name === 'CastError') {
    const message = `Resource not found`;
    error = new ErrorResponse(message, 404);
  }

  // Mongoose duplicate key
  if (err.code === 11000) {
    const message = 'Duplicate field value entered';
    error = new ErrorResponse(message, 400);
  }

  // Mongoose validation error
  if (err.name === 'ValidationError') {
    const message = Object.values(err.errors).map((val: any) => val.message);
    error = new ErrorResponse(message.join(', '), 400);
  }

  res.status(error.statusCode || 500).json({
    success: false,
    error: error.message || 'Server Error',
  });
};

export default errorHandler;
```

Create a file `src/middlewares/authMiddleware.ts`:

```typescript
import { Request, Response, NextFunction } from 'express';
import { verifyToken } from '../utils/jwtUtils';
import User from '../models/userModel';
import ErrorResponse from '../utils/errorResponse';
import mongoose from 'mongoose';

// Extend Express Request interface
declare global {
  namespace Express {
    interface Request {
      user?: any;
    }
  }
}

// Protect routes
export const protect = async (
  req: Request,
  res: Response,
  next: NextFunction
) => {
  try {
    let token;

    // Get token from Authorization header
    if (
      req.headers.authorization &&
      req.headers.authorization.startsWith('Bearer')
    ) {
      // Set token from Bearer token
      token = req.headers.authorization.split(' ')[1];
    } else if (req.cookies?.token) {
      // Set token from cookie
      token = req.cookies.token;
    }

    // Check if token exists
    if (!token) {
      return next(new ErrorResponse('Not authorized to access this route', 401));
    }

    // Verify token
    const decoded = verifyToken(token);

    // Get user from token
    req.user = await User.findById(decoded.id);

    if (!req.user) {
      return next(new ErrorResponse('Not authorized to access this route', 401));
    }

    next();
  } catch (error) {
    return next(new ErrorResponse('Not authorized to access this route', 401));
  }
};
```

Create a file `src/middlewares/uploadMiddleware.ts`:

```typescript
import multer from 'multer';
import path from 'path';
import { Request } from 'express';
import { v4 as uuidv4 } from 'uuid';

// Define storage for images
const storage = multer.diskStorage({
  destination: (req: Request, file: Express.Multer.File, cb) => {
    cb(null, path.join(__dirname, '../../uploads'));
  },
  filename: (req: Request, file: Express.Multer.File, cb) => {
    cb(null, `${uuidv4()}-${file.originalname}`);
  },
});

// Check file type
const fileFilter = (req: Request, file: Express.Multer.File, cb: any) => {
  // Allow only images
  if (file.mimetype.startsWith('image/')) {
    cb(null, true);
  } else {
    cb(new Error('Not an image! Please upload only images.'), false);
  }
};

// Initialize multer
const upload = multer({
  storage,
  fileFilter,
  limits: {
    fileSize: 5 * 1024 * 1024, // 5MB limit
  },
});

export default upload;
```

## Controllers

Now, let's implement the controllers for our API.

### Auth Controller

Create a file `src/controllers/authController.ts`:

```typescript
import { Request, Response, NextFunction } from 'express';
import User from '../models/userModel';
import ErrorResponse from '../utils/errorResponse';
import { generateToken } from '../utils/jwtUtils';

// @desc    Register user
// @route   POST /api/auth/register
// @access  Public
export const register = async (
  req: Request,
  res: Response,
  next: NextFunction
) => {
  try {
    const { name, email, password, phone, location } = req.body;

    // Check if user already exists
    const userExists = await User.findOne({ email });

    if (userExists) {
      return next(new ErrorResponse('User already exists', 400));
    }

    // Create user
    const user = await User.create({
      name,
      email,
      password,
      phone,
      location,
    });

    // Generate token
    const token = generateToken(user);

    res.status(201).json({
      success: true,
      token,
      user: {
        id: user._id,
        name: user.name,
        email: user.email,
        phone: user.phone,
        location: user.location,
        avatar: user.avatar,
      },
    });
  } catch (error) {
    next(error);
  }
};

// @desc    Login user
// @route   POST /api/auth/login
// @access  Public
export const login = async (
  req: Request,
  res: Response,
  next: NextFunction
) => {
  try {
    const { email, password } = req.body;

    // Validate email & password
    if (!email || !password) {
      return next(new ErrorResponse('Please provide an email and password', 400));
    }

    // Check for user
    const user = await User.findOne({ email }).select('+password');

    if (!user) {
      return next(new ErrorResponse('Invalid credentials', 401));
    }

    // Check if password matches
    const isMatch = await user.matchPassword(password);

    if (!isMatch) {
      return next(new ErrorResponse('Invalid credentials', 401));
    }

    // Generate token
    const token = generateToken(user);

    res.status(200).json({
      success: true,
      token,
      user: {
        id: user._id,
        name: user.name,
        email: user.email,
        phone: user.phone,
        location: user.location,
        avatar: user.avatar,
      },
    });
  } catch (error) {
    next(error);
  }
};

// @desc    Get current logged in user
// @route   GET /api/auth/me
// @access  Private
export const getMe = async (
  req: Request,
  res: Response,
  next: NextFunction
) => {
  try {
    const user = await User.findById(req.user.id);

    res.status(200).json({
      success: true,
      user,
    });
  } catch (error) {
    next(error);
  }
};

// @desc    Logout user / clear cookie
// @route   POST /api/auth/logout
// @access  Private
export const logout = async (
  req: Request,
  res: Response,
  next: NextFunction
) => {
  try {
    res.status(200).json({
      success: true,
      message: 'Logged out successfully',
    });
  } catch (error) {
    next(error);
  }
};
```

### User Controller

Create a file `src/controllers/userController.ts`:

```typescript
import { Request, Response, NextFunction } from 'express';
import User from '../models/userModel';
import ErrorResponse from '../utils/errorResponse';

// @desc    Get user profile
// @route   GET /api/users/:id
// @access  Public
export const getUserProfile = async (
  req: Request,
  res: Response,
  next: NextFunction
) => {
  try {
    const user = await User.findById(req.params.id).select('-__v');

    if (!user) {
      return next(new ErrorResponse('User not found', 404));
    }

    res.status(200).json({
      success: true,
      user,
    });
  } catch (error) {
    next(error);
  }
};

// @desc    Update user profile
// @route   PUT /api/users/profile
// @access  Private
export const updateProfile = async (
  req: Request,
  res: Response,
  next: NextFunction
) => {
  try {
    const { name, email, phone, location } = req.body;
    
    // Build update object
    const updateData: any = {};
    if (name) updateData.name = name;
    if (email) updateData.email = email;
    if (phone) updateData.phone = phone;
    if (location) updateData.location = location;
    
    // If avatar is uploaded, add it to update data
    if (req.file) {
      updateData.avatar = `/uploads/${req.file.filename}`;
    }

    const user = await User.findByIdAndUpdate(req.user.id, updateData, {
      new: true,
      runValidators: true,
    });

    res.status(200).json({
      success: true,
      user,
    });
  } catch (error) {
    next(error);
  }
};
```

## Routes

Now, let's implement the routes for our API.

### Auth Routes

Create a file `src/routes/authRoutes.ts`:

```typescript
import express from 'express';
import { register, login, getMe, logout } from '../controllers/authController';
import { protect } from '../middlewares/authMiddleware';

const router = express.Router();

router.post('/register', register);
router.post('/login', login);
router.get('/me', protect, getMe);
router.post('/logout', protect, logout);

export default router;
```

### User Routes

Create a file `src/routes/userRoutes.ts`:

```typescript
import express from 'express';
import { getUserProfile, updateProfile } from '../controllers/userController';
import { protect } from '../middlewares/authMiddleware';
import upload from '../middlewares/uploadMiddleware';

const router = express.Router();

router.get('/:id', getUserProfile);
router.put('/profile', protect, upload.single('avatar'), updateProfile);

export default router;
```

## Additional Controllers (Summary)

For completeness, you'll need to create the following additional controllers:

1. **Listing Controller** (`src/controllers/listingController.ts`):
   - Create listing
   - Get all listings
   - Get single listing
   - Update listing
   - Delete listing
   - Get user listings

2. **Category Controller** (`src/controllers/categoryController.ts`):
   - Get all categories
   - Get category by ID
   - Get subcategories

3. **Message Controller** (`src/controllers/messageController.ts`):
   - Send message
   - Get conversations
   - Get messages for a conversation

4. **Search Controller** (`src/controllers/searchController.ts`):
   - Search listings
   - Filter listings

## Connect Routes to Server

Update your `src/server.ts` to include all the routes:

```typescript
// Import routes
import authRoutes from './routes/authRoutes';
import userRoutes from './routes/userRoutes';
import listingRoutes from './routes/listingRoutes';
import categoryRoutes from './routes/categoryRoutes';
import messageRoutes from './routes/messageRoutes';
import searchRoutes from './routes/searchRoutes';
import errorHandler from './middlewares/errorMiddleware';

// Mount routes
app.use('/api/auth', authRoutes);
app.use('/api/users', userRoutes);
app.use('/api/listings', listingRoutes);
app.use('/api/categories', categoryRoutes);
app.use('/api/messages', messageRoutes);
app.use('/api/search', searchRoutes);

// Error handling middleware (should be last)
app.use(errorHandler);
```

## API Testing with Postman

To test your API, create a Postman collection with the following request groups:

1. **Auth**: 
   - Register
   - Login
   - Get Profile
   - Logout

2. **Users**:
   - Get User Profile
   - Update Profile

3. **Listings**:
   - Create Listing
   - Get Listings
   - Get Single Listing
   - Update Listing
   - Delete Listing
   - Get User Listings

4. **Categories**:
   - Get Categories
   - Get Subcategories

5. **Messages**:
   - Send Message
   - Get Conversations
   - Get Messages

6. **Search**:
   - Search Listings
   - Filter Listings

## Next Steps

With the backend API implemented, proceed to the [Frontend Development](../04-frontend-development/README.md) section to build the user interface for the OLX clone.