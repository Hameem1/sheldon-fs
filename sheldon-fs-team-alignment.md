# **Sheldon FS - Team Alignment Document**

## **1. High-Level Project Idea**

**Elevator Pitch:** Sheldon FS is an AI-powered, cross-platform file management desktop application that intelligently scans, deduplicates, and reorganizes local files. By leveraging local open-source LLMs and embeddings, it understands file context to propose optimal directory structures, ensuring complete privacy while maintaining cross-device database synchronization.

**Target Audience:** Anyone who struggles with digital clutter and wants automated, context-aware file organization across multiple operating systems without compromising their data privacy.

**Supported Platforms:**

* Windows (.exe)  
* macOS (.app)  
* Ubuntu / Linux (.deb)

## **2. Core Objectives & Scope**

**In-Scope:**

* Single-click application installation.  
* User-explicit directory selection and high-speed metadata scanning.  
* Multi-factor deduplication (primary hashing mechanism combined with file name, size, and attribute matching).  
* Generation of file embeddings using local, open-source models for supported text-rich formats (e.g., .txt, .pdf, .docx) to enable high-level operations like clustering, directory prediction, and smart renaming.  
* A dual-pane UX featuring a conversational chat interface alongside a visual file tree for reviewing and refining proposed structures.  
* Execution of proposed file movements/renames.  
* A cloud-sync mechanism to share the exact SQLite database state across the user's different desktop machines.

**Out-of-Scope (Non-Goals):**

* Cloud-based AI processing (all models must run locally to guarantee privacy).  
* Parsing and embedding of complex, non-contextual, or system files (e.g., .exe, .deb, node_modules, etc.).

## **3. Tech Stack**

**Frontend:** TypeScript (UI Framework agnostic, tailored for a chat/file-tree split view)

**Desktop Wrapper:** ElectronJS

**Database:** SQLite (with custom Cloud-Sync layer)

**AI / ML Model:** Local, open-source LLMs and embedding models (Privacy-first)

## **4. Primary User Flow**

**Step 1: Selection & Scanning:** The user launches the app and selects specific directories. The app scans these folders, collects metadata, and pushes the data to the SQLite database.

**Step 2: Analysis & Embedding:** The app identifies supported files and feeds them to the local embedding model while ignoring unsupported/system files. It concurrently runs hashing algorithms to detect exact duplicates.

**Step 3: AI Proposal & Refinement:** The model analyzes the embeddings and metadata to predict the best directory structure and suggests names for poorly named files. This is presented to the user via a visual file tree. The user converses with the LLM via the chat interface to tweak and refine the structure to their liking.

**Step 4: Execution & Safety:** Once the user is happy with the proposed state, they approve the changes. The app executes the file operations (with future iterations supporting a "dry run" visualizer and a robust "undo/revert" mechanism).
