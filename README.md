# PM2 Debugging Guide - Authorization Issues

## 1. View Real-Time Logs

```bash
# View all logs in real-time
pm2 logs

# View logs for specific app
pm2 logs your-app-name

# View only error logs
pm2 logs --err

# Clear logs and start fresh
pm2 flush
pm2 logs
```

## 2. Check Application Status

```bash
# List all running applications
pm2 list

# Show detailed info about your app
pm2 show your-app-name

# Monitor CPU and memory usage
pm2 monit
```

## 3. Restart with Fresh Logs

```bash
# Restart your application
pm2 restart your-app-name

# Or restart all apps
pm2 restart all

# Reload with zero downtime
pm2 reload your-app-name
```

## 4. Check Saved Log Files

PM2 saves logs to files. Find them:

```bash
# Default log location
cd ~/.pm2/logs

# List all log files
ls -la

# View error log
tail -f your-app-name-error.log

# View output log
tail -f your-app-name-out.log

# View last 100 lines
tail -n 100 your-app-name-out.log
```

## 5. Enable More Detailed Logging

Update your backend code temporarily to log more details:

```javascript
// In your interests route
router.patch("/hospital/:interestId", requireAuth, async (req, res) => {
  console.log("========================================");
  console.log("PATCH /hospital/:interestId called");
  console.log("Timestamp:", new Date().toISOString());
  console.log("Interest ID:", req.params.interestId);
  console.log("Request body:", req.body);
  console.log("User ID:", req.userId);
  console.log("User Role:", req.userRole);
  console.log("========================================");
  
  // ... rest of your code
});
```

Then restart:
```bash
pm2 restart your-app-name
```

## 6. Test the Endpoint Directly

Use curl to test from your VPS:

```bash
# Replace with your actual values
curl -X PATCH \
  http://localhost:YOUR_PORT/api/interests/hospital/INTEREST_ID \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{"status":"approved"}'
```

## 7. Check Environment Variables

```bash
# Show environment variables for your app
pm2 env 0  # Replace 0 with your app ID

# Or check your ecosystem file
cat ecosystem.config.js
```

## 8. Common PM2 Issues

### Issue: Old code is running
```bash
# Pull latest code
git pull

# Restart with --update-env flag
pm2 restart your-app-name --update-env

# Or delete and restart
pm2 delete your-app-name
pm2 start your-app-script.js --name your-app-name
```

### Issue: Environment variables not loading
```bash
# Start with explicit env file
pm2 start app.js --env production

# Or use ecosystem.config.js
pm2 start ecosystem.config.js
```

### Issue: Multiple instances running
```bash
# Stop all
pm2 stop all

# Delete all
pm2 delete all

# Start fresh
pm2 start your-app-script.js --name your-app-name
```

## 9. Create Ecosystem Config (Recommended)

Create `ecosystem.config.js` in your project root:

```javascript
module.exports = {
  apps: [{
    name: "your-app-name",
    script: "./server.js",
    instances: 1,
    exec_mode: "cluster",
    env: {
      NODE_ENV: "production",
      PORT: 5000
    },
    error_file: "./logs/err.log",
    out_file: "./logs/out.log",
    log_file: "./logs/combined.log",
    time: true,
    merge_logs: true
  }]
}
```

Then use:
```bash
pm2 start ecosystem.config.js
pm2 save  # Save current process list
```

## 10. Debugging Authorization Specifically

Add this to your `requireAuth` middleware:

```javascript
export const requireAuth = async (req, res, next) => {
  console.log("=== AUTH MIDDLEWARE START ===");
  console.log("Headers:", req.headers.authorization);
  
  try {
    const token = req.headers.authorization?.split(" ")[1];
    
    if (!token) {
      console.log("ERROR: No token provided");
      return res.status(401).json({ error: "No token provided" });
    }

    const decoded = await verifyToken(token);
    console.log("Token decoded:", decoded);
    
    req.userId = decoded.userId;
    
    const admin = await Admin.findOne({ clerkId: decoded.userId });
    console.log("Admin found:", admin);
    
    if (admin) {
      req.userRole = admin.role;
    } else {
      req.userRole = "user";
    }
    
    console.log("Final auth result:", {
      userId: req.userId,
      userRole: req.userRole
    });
    console.log("=== AUTH MIDDLEWARE END ===");
    
    next();
  } catch (err) {
    console.error("AUTH ERROR:", err);
    res.status(401).json({ error: "Invalid token" });
  }
};
```

## 11. Quick Diagnostic Commands

Run these in sequence and note the output:

```bash
# 1. Check if app is running
pm2 list

# 2. Watch logs in real-time
pm2 logs your-app-name --lines 50

# 3. Make the request from frontend
# (try to approve/reject an interest)

# 4. Immediately check logs again
pm2 logs your-app-name --lines 100

# 5. Check error logs specifically
cat ~/.pm2/logs/your-app-name-error.log | tail -n 50
```

## 12. Frontend Connection Check

Make sure your frontend is pointing to the correct API:

```javascript
// In your frontend config
const API_BASE = "https://your-vps-domain.com/api"; 
// or
const API_BASE = "http://your-vps-ip:PORT/api";

console.log("API_BASE:", API_BASE); // Log this to verify
```

## 13. NGINX/Reverse Proxy (if applicable)

If you're using NGINX, check your config:

```bash
# Check NGINX config
sudo nginx -t

# View NGINX error logs
sudo tail -f /var/log/nginx/error.log

# Restart NGINX
sudo systemctl restart nginx
```

---

## Start Here (Quick Checklist):

1. ✅ Run `pm2 logs` in one terminal
2. ✅ Try to approve/reject an interest from frontend
3. ✅ Look for the "AUTHORIZATION DEBUG" output
4. ✅ Share the output here

The logs will tell us exactly what's failing!
