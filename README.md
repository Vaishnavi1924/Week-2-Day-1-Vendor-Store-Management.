# Week-2-Day-1-Vendor-Store-Management.

1. Create Store Controller

Create:

backend/controllers/storeController.js

const Store = require("../models/Store");
const User = require("../models/User");

// Create Store
const createStore = async (req, res) => {
  try {
    // Only vendors can create stores
    if (req.user.role !== "vendor") {
      return res.status(403).json({
        message: "Only vendors can create a store"
      });
    }

    // Check if vendor already has a store
    if (req.user.storeId) {
      return res.status(400).json({
        message: "You already have a store"
      });
    }

    const { name, description, logo } = req.body;

    if (!name) {
      return res.status(400).json({
        message: "Store name is required"
      });
    }

    // Create store
    const store = await Store.create({
      name,
      description,
      logo,
      owner: req.user.userId
    });

    // Connect store to vendor
    await User.findByIdAndUpdate(
      req.user.userId,
      {
        storeId: store._id
      }
    );

    res.status(201).json({
      message: "Store created successfully",
      store
    });

  } catch (error) {
    res.status(500).json({
      message: error.message
    });
  }
};


// Get My Store
const getMyStore = async (req, res) => {
  try {
    if (req.user.role !== "vendor") {
      return res.status(403).json({
        message: "Only vendors can access this"
      });
    }

    if (!req.user.storeId) {
      return res.status(404).json({
        message: "You do not have a store yet"
      });
    }

    const store = await Store.findById(
      req.user.storeId
    );

    if (!store) {
      return res.status(404).json({
        message: "Store not found"
      });
    }

    res.status(200).json({
      store
    });

  } catch (error) {
    res.status(500).json({
      message: error.message
    });
  }
};


// Update My Store
const updateStore = async (req, res) => {
  try {
    if (req.user.role !== "vendor") {
      return res.status(403).json({
        message: "Only vendors can update stores"
      });
    }

    if (!req.user.storeId) {
      return res.status(404).json({
        message: "Store not found"
      });
    }

    const { name, description, logo } = req.body;

    const store = await Store.findOneAndUpdate(
      {
        _id: req.user.storeId,
        owner: req.user.userId
      },
      {
        name,
        description,
        logo
      },
      {
        new: true,
        runValidators: true
      }
    );

    if (!store) {
      return res.status(404).json({
        message: "Store not found or access denied"
      });
    }

    res.status(200).json({
      message: "Store updated successfully",
      store
    });

  } catch (error) {
    res.status(500).json({
      message: error.message
    });
  }
};


module.exports = {
  createStore,
  getMyStore,
  updateStore
};
2. Create Store Routes

Create:

backend/routes/storeRoutes.js

const express = require("express");

const {
  createStore,
  getMyStore,
  updateStore
} = require("../controllers/storeController");

const {
  protect,
  authorizeRoles
} = require("../middleware/authMiddleware");

const router = express.Router();

// Create store
router.post(
  "/",
  protect,
  authorizeRoles("vendor"),
  createStore
);

// Get vendor's own store
router.get(
  "/my-store",
  protect,
  authorizeRoles("vendor"),
  getMyStore
);

// Update vendor's own store
router.put(
  "/my-store",
  protect,
  authorizeRoles("vendor"),
  updateStore
);

module.exports = router;
3. Connect Store Routes

Open:

backend/server.js

Add:

const storeRoutes = require("./routes/storeRoutes");

Then add:

app.use("/api/stores", storeRoutes);

Your routes now are:

POST /api/stores
GET  /api/stores/my-store
PUT  /api/stores/my-store
4. Create a Store Using Postman

First, login as a vendor.

POST
http://localhost:5000/api/auth/login

Copy the JWT token from the response.

Then:

POST
http://localhost:5000/api/stores

Authorization:

Bearer Token

Paste your JWT.

Body → raw → JSON:

{
  "name": "Vaishnavi Fashion Store",
  "description": "Trendy fashion products for everyone",
  "logo": "https://example.com/logo.png"
}

Expected response:

{
  "message": "Store created successfully",
  "store": {
    "_id": "...",
    "name": "Vaishnavi Fashion Store",
    "description": "Trendy fashion products for everyone",
    "owner": "...",
    "logo": "https://example.com/logo.png",
    "isActive": true
  }
}
5. Check Your Store

Send:

GET
http://localhost:5000/api/stores/my-store

Add your JWT:

Authorization
→ Bearer Token
→ Your JWT

You'll get your store information.

6. Update Your Store

Send:

PUT
http://localhost:5000/api/stores/my-store

Body:

{
  "name": "Vaishnavi Fashion Hub",
  "description": "Quality fashion products at affordable prices",
  "logo": "https://example.com/new-logo.png"
}

The store will be updated.

7. Frontend Store Page

Now create:

frontend/src/pages/StoreManagement.jsx

import { useEffect, useState } from "react";
import API from "../services/api";

function StoreManagement() {
  const [store, setStore] = useState(null);

  const [formData, setFormData] = useState({
    name: "",
    description: "",
    logo: ""
  });

  const [message, setMessage] = useState("");

  const fetchStore = async () => {
    try {
      const response = await API.get(
        "/stores/my-store"
      );

      setStore(response.data.store);

      setFormData({
        name: response.data.store.name,
        description: response.data.store.description,
        logo: response.data.store.logo
      });

    } catch (error) {
      setMessage(
        error.response?.data?.message ||
        "Store not found"
      );
    }
  };

  useEffect(() => {
    fetchStore();
  }, []);

  const handleChange = (e) => {
    setFormData({
      ...formData,
      [e.target.name]: e.target.value
    });
  };

  const handleCreateStore = async (e) => {
    e.preventDefault();

    try {
      const response = await API.post(
        "/stores",
        formData
      );

      setStore(response.data.store);

      setMessage(
        "Store created successfully. Please login again."
      );

    } catch (error) {
      setMessage(
        error.response?.data?.message ||
        "Unable to create store"
      );
    }
  };

  const handleUpdateStore = async (e) => {
    e.preventDefault();

    try {
      const response = await API.put(
        "/stores/my-store",
        formData
      );

      setStore(response.data.store);

      setMessage(
        "Store updated successfully"
      );

    } catch (error) {
      setMessage(
        error.response?.data?.message ||
        "Unable to update store"
      );
    }
  };

  return (
    <div className="auth-container">

      <div className="auth-box">

        <h1>
          {store
            ? "Manage Your Store"
            : "Create Your Store"}
        </h1>

        <form
          onSubmit={
            store
              ? handleUpdateStore
              : handleCreateStore
          }
        >

          <input
            type="text"
            name="name"
            placeholder="Store Name"
            value={formData.name}
            onChange={handleChange}
            required
          />

          <textarea
            name="description"
            placeholder="Store Description"
            value={formData.description}
            onChange={handleChange}
            rows="5"
          />

          <input
            type="text"
            name="logo"
            placeholder="Logo URL"
            value={formData.logo}
            onChange={handleChange}
          />

          <button type="submit">
            {store
              ? "Update Store"
              : "Create Store"}
          </button>

        </form>

        <p>{message}</p>

      </div>

    </div>
  );
}

export default StoreManagement;
8. Add Store Route to React

Open:

frontend/src/App.jsx

Import:

import StoreManagement from "./pages/StoreManagement";

Then add:

<Route
  path="/store-management"
  element={
    <ProtectedRoute
      allowedRoles={["vendor"]}
    >
      <StoreManagement />
    </ProtectedRoute>
  }
/>



                    VENDOR
                      │
                      ↓
               Vendor Dashboard
                      │
                      ↓
                Store Management
                      │
            ┌─────────┴─────────┐
            ↓                   ↓
       Create Store        Update Store
            │                   │
            └─────────┬─────────┘
                      ↓
                   MongoDB




















