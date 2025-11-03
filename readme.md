

**1. Create a new Cloudflare Workers project**
```bash
npm create cloudflare@latest -- express-d1-app
cd express-d1-app
```

**2. Install Express and dependencies**
```bash
npm i express @types/express
```
Add to `wrangler.toml` for Node.js compatibility:
```toml
compatibility_flags = ["nodejs_compat"]
```

**3. Create and configure D1 database**
```bash
npx wrangler d1 create members-db
```
Add database binding in your Wrangler config:
```json
{
  "d1_databases": [
    {
      "binding": "DB",
      "database_name": "members-db"
    }
  ]
}
```

**4. Schema (`schemas/schema.sql`):**
```sql
DROP TABLE IF EXISTS members;
CREATE TABLE IF NOT EXISTS members (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL,
  email TEXT NOT NULL UNIQUE,
  joined_date TEXT NOT NULL
);

INSERT INTO members (name, email, joined_date) VALUES
  ('Alice Johnson', 'alice@example.com', '2024-01-15'),
  ('Bob Smith', 'bob@example.com', '2024-02-20'),
  ('Carol Williams', 'carol@example.com', '2024-03-10');
```
Apply the schema:
```bash
npx wrangler d1 execute members-db --file=./schemas/schema.sql
```

**5. Initialize Express in `src/index.ts`:**
```typescript
import { env } from "cloudflare:workers";
import { httpServerHandler } from "cloudflare:node";
import express from "express";

const app = express();
app.use(express.json());

app.get("/", (req, res) => {
  res.json({ message: "Express.js running on Cloudflare Workers!" });
});

app.listen(3000);
export default httpServerHandler({ port: 3000 });
```

**6. CRUD Endpoints:**
- **Read all members:**
```typescript
app.get('/api/members', async (req, res) => {
  try {
    const { results } = await env.DB.prepare('SELECT * FROM members ORDER BY joined_date DESC').all();
    res.json({ success: true, members: results });
  } catch {
    res.status(500).json({ success: false, error: 'Failed to fetch members' });
  }
});
```

- **Read by ID:**
```typescript
app.get('/api/members/:id', async (req, res) => {
  try {
    const { id } = req.params;
    const { results } = await env.DB.prepare('SELECT * FROM members WHERE id = ?').bind(id).all();
    if (results.length === 0) {
      return res.status(404).json({ success: false, error: 'Member not found' });
    }
    res.json({ success: true, member: results[0] });
  } catch {
    res.status(500).json({ success: false, error: 'Failed to fetch member' });
  }
});
```

- **Create member:**
```typescript
app.post("/api/members", async (req, res) => {
  try {
    const { name, email } = req.body;
    if (!name || !email) return res.status(400).json({ success: false, error: "Name and email are required" });
    // Email validation omitted for brevity
    const joined_date = new Date().toISOString().split("T")[0];
    const result = await env.DB.prepare("INSERT INTO members (name, email, joined_date) VALUES (?, ?, ?)")
      .bind(name, email, joined_date).run();
    if (result.success) {
      res.status(201).json({ success: true, message: "Member created successfully", id: result.meta.last_row_id });
    } else {
      res.status(500).json({ success: false, error: "Failed to create member" });
    }
  } catch (error: any) {
    if (error.message?.includes("UNIQUE constraint failed")) return res.status(409).json({ success: false, error: "Email already exists" });
    res.status(500).json({ success: false, error: "Failed to create member" });
  }
});
```

- **Update member:**
```typescript
app.put("/api/members/:id", async (req, res) => {
  try {
    const { id } = req.params;
    const { name, email } = req.body;
    if (!name && !email)
      return res.status(400).json({ success: false, error: "At least one field (name or email) is required" });
    // Build update query dynamically
    const updates: string[] = [];
    const values: any[] = [];
    if (name) { updates.push("name = ?"); values.push(name); }
    if (email) { updates.push("email = ?"); values.push(email); }
    values.push(id);
    const result = await env.DB.prepare(`UPDATE members SET ${updates.join(", ")} WHERE id = ?`).bind(...values).run();
    if (result.meta.changes === 0) {
      return res.status(404).json({ success: false, error: "Member not found" });
    }
    res.json({ success: true, message: "Member updated successfully" });
  } catch (error: any) {
    if (error.message?.includes("UNIQUE constraint failed")) return res.status(409).json({ success: false, error: "Email already exists" });
    res.status(500).json({ success: false, error: "Failed to update member" });
  }
});
```

- **Delete member:**
```typescript
app.delete("/api/members/:id", async (req, res) => {
  try {
    const { id } = req.params;
    const result = await env.DB.prepare("DELETE FROM members WHERE id = ?").bind(id).run();
    if (result.meta.changes === 0) {
      return res.status(404).json({ success: false, error: "Member not found" });
    }
    res.json({ success: true, message: "Member deleted successfully" });
  } catch {
    res.status(500).json({ success: false, error: "Failed to delete member" });
  }
});
```

**Test endpoints locally with curl** (replace port/URL as needed):
```bash
curl http://localhost:8787/api/members
curl -X POST http://localhost:8787/api/members -H "Content-Type: application/json" -d '{"name":"David Brown","email":"david@example.com"}'
curl http://localhost:8787/api/members/1
curl -X PUT http://localhost:8787/api/members/1 -H "Content-Type: application/json" -d '{"name":"Alice Cooper"}'
curl -X DELETE http://localhost:8787/api/members/4
```

***

This covers the main workflow and code snippets for building and deploying a CRUD Express.js API on Cloudflare Workers with D1 database support.[1]

[1](https://developers.cloudflare.com/workers/tutorials/deploy-an-express-app/)