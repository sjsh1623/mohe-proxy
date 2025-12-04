# Double URL Encoding Issue - Solution

## Problem

The URL received from Spring Backend is double-encoded:

```
❌ Received (Double-encoded):
https://mohe.today/image/20901_7%25EB%25B2%2588%25EA%25B0%2580%25ED%2594%25BC%25EC%259E%2590_%25EC%2598%2581%25EB%2593%25B1%25ED%258F%25AC%25EC%25A0%2590_1.jpg

✅ Expected (Single-encoded):
https://mohe.today/image/20901_7%EB%B2%88%EA%B0%80%ED%94%BC%EC%9E%90_%EC%98%81%EB%93%B1%ED%8F%AC%EC%A0%90_1.jpg
```

**Key Issue:** `%25` should be `%` - the percent signs are being encoded again.

## Root Cause

This happens when:
1. Spring encodes the filename: `7번가피자` → `7%EB%B2%88%EA%B0%80%ED%94%BC%EC%9E%90`
2. React or HTTP client encodes it again: `%EB` → `%25EB`

## Solutions

### Solution 1: Spring Should Return Decoded Filenames (RECOMMENDED)

**In Spring Backend:**
```java
// ❌ DON'T encode the filename in the response
@GetMapping("/restaurants/{id}")
public RestaurantDto getRestaurant(@PathVariable Long id) {
    Restaurant restaurant = restaurantService.findById(id);

    // Return the raw filename, NOT URL-encoded
    RestaurantDto dto = new RestaurantDto();
    dto.setImageUrl("20901_7번가피자_영등포점_1.jpg");  // Raw filename

    return dto;
}
```

**In React Frontend:**
```javascript
// React will encode it when making the request
const imageUrl = `https://mohe.today/image/${encodeURIComponent(restaurant.imageUrl)}`;

<img src={imageUrl} alt={restaurant.name} />
```

### Solution 2: Decode in React Before Using

If you can't change Spring:

```javascript
// If Spring returns encoded filename, decode it first
const decodedFilename = decodeURIComponent(restaurant.imageUrl);
const imageUrl = `https://mohe.today/image/${encodeURIComponent(decodedFilename)}`;

<img src={imageUrl} alt={restaurant.name} />
```

### Solution 3: Use Decoded URL Directly (SIMPLEST)

```javascript
// If Spring returns the full URL, decode and use as-is
const decodedUrl = decodeURIComponent(restaurant.imageUrl);

<img src={decodedUrl} alt={restaurant.name} />
```

## Verification

Test the corrected URL:
```bash
# This works ✅
curl -I "https://mohe.today/image/20901_7%EB%B2%88%EA%B0%80%ED%94%BC%EC%9E%90_%EC%98%81%EB%93%B1%ED%8F%AC%EC%A0%90_1.jpg"

# This fails ❌ (double-encoded)
curl -I "https://mohe.today/image/20901_7%25EB%25B2%2588%25EA%25B0%2580%25ED%2594%25BC%25EC%259E%2590_%25EC%2598%2581%25EB%2593%25B1%25ED%258F%25AC%25EC%25A0%2590_1.jpg"
```

## Spring Backend Example

### Current (Causing Double Encoding)
```java
@RestController
@RequestMapping("/api/restaurants")
public class RestaurantController {

    @GetMapping("/{id}/images")
    public List<String> getRestaurantImages(@PathVariable Long id) {
        List<String> images = imageService.getImagesByRestaurantId(id);

        // ❌ DON'T DO THIS - causes double encoding
        return images.stream()
            .map(filename -> {
                try {
                    return URLEncoder.encode(filename, "UTF-8");
                } catch (UnsupportedEncodingException e) {
                    return filename;
                }
            })
            .collect(Collectors.toList());
    }
}
```

### Fixed (Return Raw Filenames)
```java
@RestController
@RequestMapping("/api/restaurants")
public class RestaurantController {

    @GetMapping("/{id}/images")
    public List<String> getRestaurantImages(@PathVariable Long id) {
        // ✅ Return raw filenames - let the client encode them
        return imageService.getImagesByRestaurantId(id);
    }
}
```

## React Frontend Example

### Option A: Backend Returns Raw Filenames (BEST)
```javascript
function RestaurantImage({ restaurant }) {
  // Backend returns: "20901_7번가피자_영등포점_1.jpg"
  const imageUrl = `https://mohe.today/image/${encodeURIComponent(restaurant.imageFilename)}`;

  return <img src={imageUrl} alt={restaurant.name} />;
}
```

### Option B: Backend Returns Encoded Filenames
```javascript
function RestaurantImage({ restaurant }) {
  // Backend returns: "20901_7%EB%B2%88%EA%B0%80%ED%94%BC%EC%9E%90_%EC%98%81%EB%93%B1%ED%8F%AC%EC%A0%90_1.jpg"
  // Decode first to avoid double encoding
  const decodedFilename = decodeURIComponent(restaurant.imageFilename);
  const imageUrl = `https://mohe.today/image/${encodeURIComponent(decodedFilename)}`;

  return <img src={imageUrl} alt={restaurant.name} />;
}
```

### Option C: Backend Returns Full URLs
```javascript
function RestaurantImage({ restaurant }) {
  // Backend returns: "https://mohe.today/image/20901_7%EB%B2%88%EA%B0%80..."
  // If single-encoded, use directly
  const imageUrl = restaurant.imageUrl;

  // If double-encoded, decode once
  // const imageUrl = decodeURIComponent(restaurant.imageUrl);

  return <img src={imageUrl} alt={restaurant.name} />;
}
```

## Testing

```javascript
// Test encoding/decoding
const originalFilename = "20901_7번가피자_영등포점_1.jpg";
const encoded = encodeURIComponent(originalFilename);
const doubleEncoded = encodeURIComponent(encoded);

console.log("Original:", originalFilename);
console.log("Single encoded:", encoded);
// 20901_7%EB%B2%88%EA%B0%80%ED%94%BC%EC%9E%90_%EC%98%81%EB%93%B1%ED%8F%AC%EC%A0%90_1.jpg ✅

console.log("Double encoded:", doubleEncoded);
// 20901_7%25EB%25B2%2588%25EA%25B0%2580... ❌

// To fix double encoding:
const fixed = decodeURIComponent(doubleEncoded);
console.log("Fixed:", fixed === encoded); // true
```

## Recommendation

**The best approach:**

1. **Spring Backend**: Return raw filenames (no encoding)
   ```java
   return "20901_7번가피자_영등포점_1.jpg";
   ```

2. **React Frontend**: Encode when building the URL
   ```javascript
   const url = `https://mohe.today/image/${encodeURIComponent(filename)}`;
   ```

This keeps the data layer clean and puts encoding responsibility where it belongs - at the presentation/request layer.

## Quick Fix for Immediate Use

If you can't modify Spring right now, add this helper in React:

```javascript
// utils/imageUrl.js
export function getImageUrl(filenameOrUrl) {
  // Check if it's already a full URL
  if (filenameOrUrl.startsWith('http')) {
    // Decode once if double-encoded
    const decoded = decodeURIComponent(filenameOrUrl);
    return decoded;
  }

  // It's just a filename - decode if needed, then encode properly
  const decoded = decodeURIComponent(filenameOrUrl);
  return `https://mohe.today/image/${encodeURIComponent(decoded)}`;
}

// Usage:
<img src={getImageUrl(restaurant.imageUrl)} alt={restaurant.name} />
```
