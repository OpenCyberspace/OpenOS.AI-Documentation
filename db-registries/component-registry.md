# Component registry

Component registry is a database of all the components on-boarded by developers on the platform, the components from the components registry can be instantiated as a block by following the block specification.

**Features**:

1. Stores the components submitted by developers which can be instantiated as blocks.

2. Provides REST and GraphQL interface for querying, filtering and discovery of components.

3. Provides a well defined schema to upload component metadata, input and output schema templates, management commands schema templates , settings and parameters schema templates, default settings and parameters etc.

## Schema of component registry:

```js
// Sub-schema for componentId
const componentIdSchema = new mongoose.Schema({
  name: { type: String, required: true }, // Component name
  version: { type: String, required: true }, // Component version
  releaseTag: { type: String, required: true }  // Component release tag
}, { _id: false })

// Sub-schema for containerRegistryInfo
const containerRegistryInfoSchema = new mongoose.Schema({
  containerImage: { type: String, required: true }, // Image path in registry
  containerRegistryId: { type: String, required: true }, // Registry identifier
  containerImageMetadata: { type: mongoose.SchemaTypes.Mixed, default: {} }, // Arbitrary metadata (description, author, etc.)
  componentMode: { type: String, enum: ['aios', 'third_party'], required: true }, // Component origin

  // init container: specified only if the componentMode = "third_party"
  initContainer: { type: mongoose.SchemaTypes.Mixed, default: {} }

}, { _id: false })

const componentSchema = new mongoose.Schema({
  createdAt: { type: Date, default: Date.now }, // Creation timestamp (sys generated)
  lastModifiedAt: { type: Date, default: Date.now }, // Last modified timestamp (sys generated)

  componentId: { type: componentIdSchema, required: true }, // Structured component identity
  componentType: { type: String, required: true },

  // Unique URI (assigned internally - system generated) - {<componentType>.<componentId.name>:<componentId.version>-<componentId.releaseTag}
  componentURI: { type: String, required: true, index: true }, 

  containerRegistryInfo: { type: containerRegistryInfoSchema, required: false }, // Container registry information

  componentMetadata: { type: mongoose.SchemaTypes.Mixed, default: {} }, // Custom metadata

  componentInitData: { type: mongoose.SchemaTypes.Mixed, default: {} }, // Initial data/config needed to start
  componentInputProtocol: { type: mongoose.SchemaTypes.Mixed, default: {} }, // Input protocol or data schema
  componentOutputProtocol: { type: mongoose.SchemaTypes.Mixed, default: {} }, // Output protocol or data schema
  policies: { type: mongoose.SchemaTypes.Mixed, default: {} }, // Attached policies

  componentManagementCommandsTemplate: { type: mongoose.SchemaTypes.Mixed, default: {} }, // List of management commands supported
  componentInitSettings: { type: mongoose.SchemaTypes.Mixed, default: {} }, // Runtime settings or flags
  componentParameters: { type: mongoose.SchemaTypes.Mixed, default: {} }, // Param list for inference or processing

  componentInitSettingsProtocol: { type: mongoose.SchemaTypes.Mixed, default: {} }, // Protocol/template of init settings
  componentInitParametersProtocol: { type: mongoose.SchemaTypes.Mixed, default: {} }, // Protocol/template of init params

  tags: [{ type: String }] // Free-form list of tags for search/filtering
})
```

**Example**:

```json
{
        "componentId": {
          "name": "object-detector",
          "version": "2.0.0",
          "releaseTag": "stable"
        },
        "componentURI": "model.object-detector:2.0.0-stable",
        "componentType": "model",

        "containerRegistryInfo": {
          "containerImage": "registry.ai-platform.com/models/object-detector:2.0.0",
          "containerRegistryId": "ai-platform-registry",
          "containerImageMetadata": {
            "author": "vision-team",
            "description": "YOLO-based real-time object detection model"
          },
          "componentMode": "aios"
        },

        "componentMetadata": {
          "usecase": "real-time object detection",
          "framework": "PyTorch",
          "hardware": "GPU"
        },

        "componentInitData": {
          "weights_path": "/models/yolov5s.pt",
          "device": "cuda"
        },

        "componentInputProtocol": {
          "image": {
            "type": "string",
            "description": "Input image encoded as Base64",
            "pattern": "^data:image/.*;base64,"
          },
          "image_shape": {
            "type": "array",
            "description": "Height and width of the input image",
            "max_length": 2,
            "items": {
              "type": "number",
              "min": 1
            }
          }
        },

        "componentOutputProtocol": {
          "detections": {
            "type": "array",
            "description": "Detected objects",
            "items": {
              "type": "object",
              "properties": {
                "class": { "type": "string" },
                "confidence": { "type": "number", "min": 0.0, "max": 1.0 },
                "bbox": {
                  "type": "array",
                  "description": "Bounding box [x_min, y_min, x_max, y_max]",
                  "max_length": 4,
                  "items": { "type": "number" }
                }
              }
            }
          },
          "processing_time_ms": {
            "type": "number",
            "description": "Inference time in milliseconds"
          }
        },

        "componentInitParametersProtocol": {
          "threshold": {
            "type": "number",
            "description": "Confidence threshold for filtering predictions",
            "min": 0.0,
            "max": 1.0
          },
          "top_k": {
            "type": "number",
            "description": "Return top-K detections per image",
            "min": 1,
            "max": 50
          }
        },

        "componentInitSettingsProtocol": {
          "enable_tracing": {
            "type": "boolean",
            "description": "Enable performance tracing logs"
          },
          "batch_size": {
            "type": "number",
            "description": "Batch size to be used during inference",
            "min": 1,
            "max": 32
          }
        },

        "policies": {
          "resource_affinity": {
            "nodeType": "gpu",
            "minMemory": "4GB"
          }
        },

        "componentManagementCommandsTemplate": {
          "restart": {
            "description": "Restart the model service",
            "args": {}
          },
          "reload_weights": {
            "description": "Reload model weights from disk",
            "args": {
              "weights_path": {
                "type": "string",
                "required": true
              }
            }
          }
        },

        "componentParameters": {
          "threshold": 0.4,
          "top_k": 10
        },

        "componentInitSettings": {
          "enable_tracing": true,
          "batch_size": 4
        },

        "tags": ["vision", "object-detection", "yolo", "realtime"]
}
```

---

### Component APIs json:


### **Endpoint:** `/api/registerComponent (used internally by the parser)`  
**Method:** `POST`  
**Description:**  
Registers a new component. Automatically generates `componentURI` using `componentType`, `componentId.name`, `componentId.version`, and `componentId.releaseTag`. Fails if a component with the same `componentURI` already exists.

**Example curl Command:**
```bash
curl -X POST http://<server-url>/api/registerComponent \
  -H "Content-Type: application/json" \
  -d @./component.json
```

**Note:** The file `component.json` should match the schema (see your Mongoose model). Do not include `componentURI`—it's generated by the server.

---

### **Endpoint:** `/api/unregisterComponent`  
**Method:** `POST`  
**Description:**  
Deletes a component using its `componentURI`.

**Example curl Command:**
```bash
curl -X POST http://<server-url>/api/unregisterComponent \
  -H "Content-Type: application/json" \
  -d '{
    "uri": "parser.tokenizer:1.0-latest"
  }'
```

---

### **Endpoint:** `/api/updateComponent`  
**Method:** `POST`  
**Description:**  
Updates an existing component's fields using a MongoDB-style update query.

**Example curl Command:**
```bash
curl -X POST http://<server-url>/api/updateComponent \
  -H "Content-Type: application/json" \
  -d '{
    "uri": "parser.tokenizer:1.0-latest",
    "data": {
      "$set": {
        "componentMetadata.description": "Updated tokenizer component"
      }
    }
  }'
```

---

### **Endpoint:** `/api/addMetadata`  
**Method:** `POST`  
**Description:**  
Adds or replaces the `metadata` field of a component. The value is validated before saving.

**Example curl Command:**
```bash
curl -X POST http://<server-url>/api/addMetadata \
  -H "Content-Type: application/json" \
  -d '{
    "uri": "parser.tokenizer:1.0-latest",
    "data": {
      "author": "AI Team",
      "build": "v1.2"
    }
  }'
```

---

### **Endpoint:** `/api/updateMetadata`  
**Method:** `POST`  
**Description:**  
Performs an update on the `metadata` field using MongoDB update operators.

**Example curl Command:**
```bash
curl -X POST http://<server-url>/api/updateMetadata \
  -H "Content-Type: application/json" \
  -d '{
    "uri": "parser.tokenizer:1.0-latest",
    "data": {
      "$set": {
        "metadata.author": "AIOS Team"
      }
    }
  }'
```

---

### **Endpoint:** `/api/getByType`  
**Method:** `POST`  
**Description:**  
Finds components by matching `componentType` using regex.

**Example curl Command:**
```bash
curl -X POST http://<server-url>/api/getByType \
  -H "Content-Type: application/json" \
  -d '{
    "typeString": "parser"
  }'
```

---

### **Endpoint:** `/api/getByURI`  
**Method:** `POST`  
**Description:**  
Finds components where `componentURI` matches the provided string as regex.

**Example curl Command:**
```bash
curl -X POST http://<server-url>/api/getByURI \
  -H "Content-Type: application/json" \
  -d '{
    "uriString": "parser.tokenizer:1.0-latest"
  }'
```

---

### **Endpoint:** `/api/query`  
**Method:** `POST`  
**Description:**  
Performs a MongoDB-style query on the components collection. Accepts any standard MongoDB filter syntax.

**Example curl Command:**
```bash
curl -X POST http://<server-url>/api/query \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "tags": { "$in": ["production"] },
      "componentId.version": "1.0"
    }
  }'
```

---

### GraphQL API:


### **Endpoint:** `/gql`  
**Method:** `POST` (GraphQL queries), `GET` (for accessing GraphiQL playground)  
**Description:**  
GraphQL endpoint to query component data from MongoDB using a read-only schema. Mutations are **not supported**. This endpoint is based on the Mongoose `Component` model using `graphql-compose-mongoose` and exposes only query resolvers.

- Schema: `schemaBuilders`
- Resolvers: `queryResolvers`
- UI: GraphiQL is enabled for interactive query testing

---

## Usage

### GraphiQL UI Access

Visit in your browser:

```
http://<server-url>/gql
```

---

### Example Query: Find All Components by Type

```graphql
query {
  componentMany(filter: { componentType: "object-det" }) {
    componentURI
    componentId {
      name
      version
      releaseTag
    }
    componentMetadata
  }
}
```

---

### Example Query: Get Component by ID

```graphql
query {
  componentById(_id: "661d44aa99a3a21f25ac9c70") {
    componentURI
    componentType
    componentId {
      name
      version
      releaseTag
    }
  }
}
```

---

### Example curl Command

```bash
curl -X POST http://<server-url>/gql \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query { componentMany(filter: { componentType: \"parser\" }) { componentURI componentType } }"
  }'
```

---
