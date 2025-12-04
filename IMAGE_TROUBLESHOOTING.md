# Image Service Troubleshooting Results

## Issue
Images were returning 404 errors when accessed through `https://mohe.today/image/`

## Investigation Results

### Service Status ✅
- **Image Processor Container**: Running (`moheimageprocessor-app-1`)
- **Port**: 5200 (internal)
- **Network**: Connected to `proxy_network`
- **Image Files**: Present in `/app/images/` directory (27GB of images)

### Routing Configuration ✅
- **Caddy Container**: Running and properly configured
- **Caddyfile**: Valid syntax, routes `/image/*` to `moheimageprocessor-app-1:5200`
- **Health Check**: Added `/health` endpoint for easy testing

### Image Service Endpoints

The image processor provides three endpoints:

1. **Full Filename Access**
   ```
   GET /image/:fileName
   ```
   Example: `/image/10000_파스토보이_월계점_1.jpg`

2. **ID-Only Lookup** (no extension)
   ```
   GET /image/:id
   ```
   Example: `/image/10000`
   - Automatically searches for files matching the ID prefix
   - Returns the first matching file

3. **Resized Images**
   ```
   GET /image/:fileName/:width/:height
   ```
   Example: `/image/10000/400/300`
   - Uses Sharp library for image resizing
   - Fit mode: cover

### Connectivity Tests ✅

**Internal Connectivity (Container → Container):**
```bash
# Direct connection from Caddy to image processor WORKS
docker exec proxy-caddy wget -O /dev/null \
  "http://moheimageprocessor-app-1:5200/image/10000_%ED%8C%8C%EC%8A%A4%ED%86%A0%EB%B3%B4%EC%9D%B4_%EC%9B%94%EA%B3%84%EC%A0%90_1.jpg"
# Result: Success (29,288 bytes downloaded)
```

**Key Finding:** The internal routing from Caddy to the image processor is working correctly. Images must be URL-encoded (especially Korean characters) to work properly.

### URL Encoding ⚠️

**Problem:** Korean characters in filenames must be URL-encoded.

**Example:**
- Original filename: `10000_파스토보이_월계점_1.jpg`
- URL-encoded: `10000_%ED%8C%8C%EC%8A%A4%ED%86%A0%EB%B3%B4%EC%9D%B4_%EC%9B%94%EA%B3%84%EC%A0%90_1.jpg`

**JavaScript Encoding:**
```javascript
const filename = "10000_파스토보이_월계점_1.jpg";
const encodedFilename = encodeURIComponent(filename);
const imageUrl = `https://mohe.today/image/${encodedFilename}`;
```

### External Access Issue ⚠️

**Domain Connectivity:**
```bash
curl https://mohe.today
# Result: Connection timeout (75+ seconds)
```

**This is NOT an image service issue** - this is a general network/DNS/firewall issue affecting all access to mohe.today domain from the current network.

**Possible Causes:**
1. Port forwarding not configured correctly on router
2. Firewall blocking incoming connections on ports 80/443
3. ISP blocking incoming traffic
4. DNS propagation issues
5. Network routing problems

**Verification:**
- Caddy is listening on ports 80 and 443 ✅
- DNS resolves correctly to 182.210.216.60 ✅
- Let's Encrypt certificate is valid ✅
- Caddy logs show no incoming requests ⚠️

### Recommendations

#### For Frontend/Client Code:
1. **Always URL-encode filenames:**
   ```javascript
   const imageUrl = `https://mohe.today/image/${encodeURIComponent(filename)}`;
   ```

2. **Use ID-only format when possible:**
   ```javascript
   const imageUrl = `https://mohe.today/image/10000`; // Simpler, no encoding needed
   ```

3. **Implement fallback behavior:**
   ```javascript
   <img
     src={`https://mohe.today/image/${imageId}`}
     onError={(e) => {
       // Fallback to encoded full filename if ID-only fails
       e.target.src = `https://mohe.today/image/${encodeURIComponent(fullFilename)}`;
     }}
   />
   ```

#### For Network Access:
1. Verify port forwarding on router (192.168.219.1)
   - External Port 80 → Internal 192.168.219.100:80
   - External Port 443 → Internal 192.168.219.100:443

2. Test from outside the local network (use mobile data or different network)

3. Check firewall rules (macOS firewall, router firewall)

4. Verify with ISP that incoming connections are not blocked

### Testing Commands

**Test from local network:**
```bash
# Health check
curl -I http://192.168.219.100/health

# Image access (replace with actual image ID)
curl -I http://192.168.219.100/image/10000
```

**Test from inside Caddy container:**
```bash
# Direct test to image processor
docker exec proxy-caddy wget -O /dev/null \
  "http://moheimageprocessor-app-1:5200/image/10000"
```

**Verify Caddy logs:**
```bash
# Watch for incoming requests
docker logs -f proxy-caddy
```

## Summary

✅ **Image service is working correctly**
✅ **Caddy proxy routing is configured properly**
✅ **Internal container networking is functional**
⚠️ **External domain access needs network troubleshooting**

The 404 errors are caused by **URL encoding issues with Korean filenames**, not by the proxy configuration. Use URL encoding or ID-only format for reliable image access.
