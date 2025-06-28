# Software-Engineering-Intern-Assignment
# Data Alchemist: AI-Powered Resource Allocation Configurator

**Data Alchemist** helps teams transform messy spreadsheets into clean, validated, rule‑ready data for resource allocation — powered by AI and built with Next.js + TypeScript.

---

**Features at a glance**
- Upload CSV/XLSX for clients, workers, and tasks
- AI header mapper: fixes misnamed/missing columns
- Real‑time validations & inline edits (missing fields, broken JSON, unknown IDs, circular references, etc.)
- Natural language search: "Show tasks >2 phases with phase 3 preferred"
- Natural language to rule converter: "Tasks T12 & T14 must always run together"
- Visual rule builder UI
- Prioritization sliders & presets (e.g. Cost Saver, Balanced)
- Export: clean data + rules.json ready for downstream allocator
- AI suggestions: detect conflicting rules, overloaded workers & auto‑fix broken data

---

**Tech stack**
- Next.js (TypeScript)
- shadcn/ui + Tailwind CSS
- TanStack Table (editable grid)
- SheetJS (CSV/XLSX parsing)
- OpenAI / local LLM (o4‑mini) for AI/NLP
- Zustand + React context (state & rules)

---

**Folder structure**
```plaintext
/pages
  index.tsx             ← Upload, data grid, validations
  rules.tsx             ← Rule builder UI
  prioritize.tsx        ← Prioritization UI
/components
  DataGrid.tsx
  UploadSection.tsx
  ValidationPanel.tsx
  RuleBuilder.tsx
  NaturalLanguageBar.tsx
/lib
  aiParser.ts           ← AI header mapping
  validators.ts         ← Core + AI validators
  ruleEngine.ts         ← NLP → JSON rule converter
/public/samples
  clients.csv, workers.csv, tasks.csv

Folder structure
bash
Copy
Edit
/pages
  index.tsx
  rules.tsx
  prioritize.tsx
/components
  UploadSection.tsx
  DataGrid.tsx
  ValidationPanel.tsx
  RuleBuilder.tsx
  NaturalLanguageBar.tsx
/lib
  aiParser.ts
  validators.ts
  ruleEngine.ts
/public/samples
  clients.csv
  workers.csv
  tasks.csv
 /pages/index.tsx
Upload files, show editable grids, validations, AI search

tsx
Copy
Edit
// pages/index.tsx
import { useState } from "react";
import UploadSection from "@/components/UploadSection";
import DataGrid from "@/components/DataGrid";
import ValidationPanel from "@/components/ValidationPanel";
import NaturalLanguageBar from "@/components/NaturalLanguageBar";
import { validateAll } from "@/lib/validators";
import { parseFilesWithAI } from "@/lib/aiParser";

export default function HomePage() {
  const [data, setData] = useState<{clients: any[], workers: any[], tasks: any[]}>({
    clients: [], workers: [], tasks: []
  });
  const [errors, setErrors] = useState<string[]>([]);

  async function handleFiles(files: FileList) {
    const parsed = await parseFilesWithAI(files);
    setData(parsed);
    const errs = validateAll(parsed);
    setErrors(errs);
  }

  return (
    <div className="p-4 space-y-4">
      <h1 className="text-2xl font-bold">🧪 Data Alchemist</h1>
      <UploadSection onFilesUploaded={handleFiles} />
      <NaturalLanguageBar data={data} setData={setData} />
      <ValidationPanel errors={errors} />
      <DataGrid title="Clients" data={data.clients} setData={(d)=>setData(p=>({...p, clients:d}))} />
      <DataGrid title="Workers" data={data.workers} setData={(d)=>setData(p=>({...p, workers:d}))} />
      <DataGrid title="Tasks" data={data.tasks} setData={(d)=>setData(p=>({...p, tasks:d}))} />
    </div>
  );
}

/components/UploadSection.tsx
tsx
Copy
Edit
// components/UploadSection.tsx
export default function UploadSection({ onFilesUploaded }: { onFilesUploaded: (files: FileList) => void }) {
  return (
    <div>
      <input type="file" accept=".csv,.xlsx" multiple onChange={e => e.target.files && onFilesUploaded(e.target.files)} />
    </div>
  );
}

/components/DataGrid.tsx
Simple editable table with TanStack Table

tsx
Copy
Edit
// components/DataGrid.tsx
import { useState } from "react";

export default function DataGrid({ title, data, setData }:
 { title: string, data: any[], setData: (d:any[])=>void }) {
  const [localData, setLocalData] = useState(data);

  return (
    <div>
      <h2 className="font-semibold">{title}</h2>
      <table className="border text-sm">
        <thead>
          <tr>
            {Object.keys(localData[0]||{}).map(k=><th key={k} className="border">{k}</th>)}
          </tr>
        </thead>
        <tbody>
          {localData.map((row, i)=>(
            <tr key={i}>
              {Object.entries(row).map(([k,v])=>
                <td key={k} className="border">
                  <input
                    value={v}
                    onChange={e=>{
                      const copy = [...localData];
                      copy[i][k]=e.target.value;
                      setLocalData(copy);
                      setData(copy);
                    }}
                    className="w-24"
                  />
                </td>
              )}
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  )
}

 /components/ValidationPanel.tsx
tsx
Copy
Edit
// components/ValidationPanel.tsx
export default function ValidationPanel({ errors }:{errors:string[]}) {
  if (!errors.length) return <p className="text-green-600">No validation errors</p>
  return (
    <div className="text-red-600">
      <p> Validation errors:</p>
      <ul>{errors.map((e,i)=><li key={i}>{e}</li>)}</ul>
    </div>
  )
}

/components/NaturalLanguageBar.tsx
Simple text input to do NLP search/modifications

tsx
Copy
Edit
// components/NaturalLanguageBar.tsx
import { runNLQuery } from "@/lib/aiParser";

export default function NaturalLanguageBar({ data, setData }:{data:any, setData:(d:any)=>void}) {
  async function handleQuery(q:string) {
    const newData = await runNLQuery(q, data);
    setData(newData);
  }

  return (
    <input type="text" placeholder="Ask: show tasks >2 phases..."
      onKeyDown={e=>{if(e.key==='Enter')handleQuery(e.currentTarget.value)}}
      className="border px-2 py-1 w-full"
    />
  );
}

/lib/aiParser.ts
AI header mapping & NLP search

ts
Copy
Edit
// lib/aiParser.ts
import * as XLSX from 'xlsx';

export async function parseFilesWithAI(files: FileList) {
  const result:{clients:any[], workers:any[], tasks:any[]} = {clients:[], workers:[], tasks:[]};
  for(const file of Array.from(files)){
    const data = await file.arrayBuffer();
    const wb = XLSX.read(data);
    const json = XLSX.utils.sheet_to_json(wb.Sheets[wb.SheetNames[0]]);
    // naive guess: name contains 'client', 'worker', or 'task'
    if(file.name.toLowerCase().includes('client')) result.clients = json;
    else if(file.name.toLowerCase().includes('worker')) result.workers = json;
    else if(file.name.toLowerCase().includes('task')) result.tasks = json;
  }
  return result;
}

export async function runNLQuery(q:string, data:any) {
  console.log("Would send to AI:", q);
  return data; // stub: return unchanged
}

/lib/validators.ts
Hard-coded sample validations

ts
Copy
Edit
// lib/validators.ts
export function validateAll(data:{clients:any[], workers:any[], tasks:any[]}) {
  const errs:string[] = [];
  if(data.clients.some(c=>!c.PriorityLevel)) errs.push("Missing PriorityLevel in some clients");
  if(data.tasks.some(t=>t.Duration<1)) errs.push("Task duration <1 found");
  return errs;
}

/pages/rules.tsx
Rule builder page

tsx
Copy
Edit
// pages/rules.tsx
import RuleBuilder from "@/components/RuleBuilder";

export default function RulesPage() {
  return (
    <div className="p-4">
      <h1 className="text-xl font-bold">Rule Builder</h1>
      <RuleBuilder />
    </div>
  )
}
/components/RuleBuilder.tsx
tsx
Copy
Edit
// components/RuleBuilder.tsx
export default function RuleBuilder() {
  return (
    <div>
      <p>UI for adding co-run, slot-restriction etc.</p>
      <button className="border px-2 py-1">+ Add Co-Run Rule</button>
    </div>
  );
}

/pages/prioritize.tsx
Weights UI

tsx
Copy
Edit
// pages/prioritize.tsx
export default function PrioritizePage() {
  return (
    <div className="p-4">
      <h1 className="text-xl font-bold"> Prioritization & Weights</h1>
      <div>
        <label>PriorityLevel weight
          <input type="range" min="0" max="10" defaultValue="5" />
        </label>
      </div>
    </div>
  )
}

