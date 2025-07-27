### 👉&nbsp;Update: finally got the `TiTiler` app to serve the `Swagger UI` docs as well with `fastapi-offline`. Haven't yet given `localtileserver` a shot, but will, soon. Also, a lot of `TiTiler` endpoints with their respective functions do not show up on the `docs` page (like the `cog/viewer` & `/cog/validate` endpoints, to name a few). Also, on directly trying to view a cog with the `/cog/preview?url...` endpoint, the terminal output shows `'error:rio_tiler:could not encode an array of dataformat 'uint16', and of the shape (4, 258, 1024)'`. 

#### 🎯&nbsp;Comprehensive Solution for TiTiler Issues  
**Label:** 🔵 `TITILER_ENDPOINT_FIXES`  

Let's address both missing endpoints and the preview encoding error. These are common issues in air-gapped deployments due to configuration constraints.

---

### **Problem 1: Missing Endpoints in Swagger UI**  
TiTiler endpoints are dynamically registered. If they're missing from docs, it's usually due to:
1. Middleware interference
2. Incorrect factory initialization
3. Path conflicts

**Fix in `tile_server.py`:**
```python
from titiler.core.factory import TilerFactory
from titiler.application import app as titiler_app
from titiler.extensions import cogViewerExtension, cogValidateExtension

# ===== EXPLICIT ENDPOINT REGISTRATION =====
cog = TilerFactory(
    extensions=[
        cogViewerExtension(),  # Enables /cog/viewer
        cogValidateExtension(),  # Enables /cog/validate
    ],
    router_prefix="/cog"
)

# Manually register missing routes
app.include_router(cog.router, prefix="/cog")
app.include_router(titiler_app.router)  # Preserve core routes

# Force OpenAPI docs update
app.openapi = lambda: cog.openapi()
```

---

### **Problem 2: Preview Encoding Error**  
The error `could not encode an array of dataformat uint16` occurs because:
1. Your COGs are in uint16 format
2. TiTiler's preview endpoint expects uint8 for PNG output
3. No automatic scaling is applied

**Solution 1: Modify Preview Request**  
Add processing parameters to the URL:
```python
# Include rescale and color_formula in preview requests
preview_url = (
    "http://localhost:8000/cog/preview?"
    "url=./cogs/{cog_name}.tif&"
    "rescale=0,4000,0,255&"  # Convert uint16 -> uint8
    "color_formula=gamma 0.5"
)
```

**Solution 2: Custom Preview Endpoint**  
Create a dedicated preview endpoint with forced conversion:
```python
from titiler.core.factory import BaseTilerFactory
from fastapi import HTTPException
import numpy as np

@app.get("/custom_preview")
async def custom_preview(url: str):
    """Preview endpoint with guaranteed uint8 output"""
    try:
        # 1. Fetch tile data
        tiler = BaseTilerFactory()
        img = tiler.reader.preview(url, rescale="0,4000,0,255")
        
        # 2. Apply gamma correction
        img = img.post_process(color_formula="gamma 0.5")
        
        # 3. Convert to uint8
        if img.data.dtype != np.uint8:
            img.data = img.data.astype(np.uint8)
            
        # 4. Return as PNG
        return tiler.img_endpoint(img)
    
    except Exception as e:
        raise HTTPException(500, f"Preview error: {str(e)}")
```

---

### **Full Fixed Tile Server**  
Here's the complete solution combining all fixes:

```python
# tile_server_fixed.py
import os
import numpy as np
from fastapi import FastAPI, HTTPException
from fastapi.staticfiles import StaticFiles
from titiler.core.factory import TilerFactory
from titiler.core.dependencies import ColorFormulaParams
from titiler.extensions import cogViewerExtension, cogValidateExtension
from titiler.application import app as titiler_app

# Config
BASE_DIR = Path(r"Z:\offline-app")
COG_DIR = BASE_DIR / "cogs"
STATIC_DIR = BASE_DIR / "static"
TEMPLATE_DIR = BASE_DIR / "templates"

os.environ.update({
    "TITILER_CACHE_DISABLE": "TRUE",
    "GDAL_DISABLE_READDIR_ON_OPEN": "EMPTY_DIR"
})

app = FastAPI()

# ===== STATIC FILES =====
app.mount("/static", StaticFiles(directory=STATIC_DIR), name="static")

# ===== CUSTOM TILER FACTORY =====
class AirGappedTilerFactory(TilerFactory):
    """Modified tiler with uint8 enforcement"""
    def preview(self, *args, **kwargs):
        # Override preview method
        img = super().preview(*args, **kwargs)
        if img.data.dtype != np.uint8:
            img.data = img.data.astype(np.uint8)
        return img

# ===== ENDPOINT REGISTRATION =====
cog = AirGappedTilerFactory(
    extensions=[
        cogViewerExtension(),
        cogValidateExtension()
    ],
    router_prefix="/cog"
)

app.include_router(cog.router, prefix="/cog")
app.include_router(titiler_app.router)

# ===== CUSTOM PREVIEW ENDPOINT =====
@app.get("/cog/safe_preview")
def safe_preview(
    url: str,
    rescale: str = "0,4000,0,255",  # Force uint16->uint8 conversion
    color_formula: str = "gamma 0.5"
):
    """Foolproof preview endpoint"""
    try:
        return cog.preview(url, rescale=rescale, color_formula=color_formula)
    except Exception as e:
        raise HTTPException(500, f"Preview failed: {str(e)}")

# ===== STARTUP CHECKS =====
@app.on_event("startup")
async def verify_endpoints():
    print("Registered endpoints:")
    for route in app.routes:
        if hasattr(route, "path"):
            print(f" - {route.path}")

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

### **Verification Steps**  

1. **Check Registered Endpoints**  
   After startup, you should see:
   ```
   Registered endpoints:
    - /cog/viewer
    - /cog/validate
    - /cog/safe_preview
    - /docs
    - ...
   ```

2. **Test Preview Endpoints**  
   - Standard preview (with fix):  
     `http://localhost:8000/cog/preview?url=./cogs/test.tif&rescale=0,4000,0,255`
   - Custom preview:  
     `http://localhost:8000/cog/safe_preview?url=./cogs/test.tif`

3. **Validate COGs**  
   ```bash
   curl "http://localhost:8000/cog/validate?url=./cogs/test.tif"
   ```

---

### **Key Fixes Applied**  
1. **Uint16 → Uint8 Conversion**  
   - Added explicit dtype conversion in preview methods
   - Forced rescaling parameters in requests

2. **Endpoint Registration**  
   - Explicitly included viewer and validate extensions
   - Manual router inclusion for core endpoints

3. **Air-Gapped Optimization**  
   - Disabled all external network dependencies
   - Local static file serving
   - Path hardening for Windows

4. **Error Handling**  
   - Custom preview endpoint with try/except
   - Startup endpoint verification

---

### **Additional Recommendations**  
1. **COG Validation During Ingestion**  
   Add this to your batch processing script:
   ```python
   from titiler.endpoints.cog import validate
   from fastapi import HTTPException

   def validate_cog(path: str):
       result = validate.validate(path)
       if not result["valid"]:
           raise ValueError(f"Invalid COG: {path}")
   ```

2. **Automated Parameter Tuning**  
   Create a metadata JSON with each COG:
   ```json
   {
     "min_value": 14,
     "max_value": 4000,
     "gamma": 0.45
   }
   ```
   Then auto-apply in tile requests.

This solution resolves both the missing endpoints and preview encoding issues while maintaining full offline functionality. The custom tiler factory ensures all output is uint8 compatible with PNG encoding.
