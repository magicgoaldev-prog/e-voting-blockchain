# E-Voting on Blockchain

A full-stack decentralized voting application that combines a React frontend, Express/MongoDB backend, and a Solidity smart contract deployed via Hardhat. Organizers manage elections off-chain; vote submission and tallying are recorded on-chain through MetaMask and ethers.js.

---

## Prerequisites

| Requirement | Version / Notes |
|-------------|-----------------|
| Node.js | 16.x or later |
| npm | Bundled with Node.js |
| MetaMask | Browser extension with a test network configured (e.g. Polygon Mumbai) |
| MongoDB | Atlas URI or local instance |

---

## Local Setup

### 1. Environment configuration

Copy the sample environment file and fill in the required values:

```bash
cp .env.sample .env
```

```env
PORT=5000
MONGO_URL=<your-mongodb-connection-string>
EMAIL=<smtp-email>
EMAILPASSWORD=<smtp-password>
```

### 2. Backend API (required for Problems 2 and 3)

From the repository root:

```bash
npm install
npm run dev
```

The API listens on `http://localhost:5000` and exposes routes under `/api/auth`.

### 3. Frontend (required for the assessment)

```bash
cd client
npm install
npm start
```

The React app runs at `http://localhost:3000`.

**For the assessment, run both the backend (step 2) and the frontend.** Problem 1 focuses on Web3 wiring and can be explored with the frontend alone, but Problems 2 and 3 load election data and authenticate voters through the API (`/api/auth/...`), so the backend must be running.

Point the client at your local API by updating `client/src/Data/Variables.jsx`:

```js
export const serverLink = "http://localhost:5000/api/auth/";
```

Ensure the contract address in `client/src/utils/contractInstance.js` matches your deployed `Voting` contract on the network configured in MetaMask.

### 4. Smart contract (optional for local testing)

```bash
cd hardhat
npm install
npx hardhat compile
npx hardhat run scripts/deploy.js --network <your-network>
```

Update the exported `contractAddress` in `client/src/utils/contractInstance.js` with the address returned by the deploy script.

---

## Developer Assessment

Complete all three problems below **with the backend and frontend running** (see steps 2–3 above). Answers that rely only on reading source code, without verifying behavior in the browser or Network tab, will not be accepted.

| Problem | Frontend | Backend |
|---------|----------|---------|
| Problem 1 — Web3 integration audit | Required | Optional |
| Problem 2 — Vote transaction trace | Required | Required |
| Problem 3 — FE/BE API integration audit | Required | Required |

**Submission format:** For each problem, provide a short written response (150–300 words) including file paths, function names, and observations confirmed in the running application.

---

### Problem 1 — Frontend & Web3 Integration Audit

**Objective:** Verify that the React client is correctly wired to the blockchain layer.

**Steps:**

1. Start the backend and frontend as described above.
2. Open `http://localhost:3000` in a browser with MetaMask installed.
3. Trigger a wallet connection from the UI (or any action that calls `contractInstance()`).
4. Repeat step 3 in a browser **without** MetaMask (or with the extension disabled).

**Deliverables:**

- Identify the utility file that creates the ethers.js `Web3Provider` and `Contract` instance. State the exported contract address and where the ABI is sourced from.
- Describe exactly what happens in the UI when MetaMask is unavailable.
- Confirm which wallet address the app uses as `msg.sender` when a transaction is submitted.

---

### Problem 2 — End-to-End Vote Transaction Trace

**Objective:** Trace a vote from the React UI to the Solidity contract and evaluate the integration.

**Steps:**

1. With the backend and frontend running, navigate to `/election` and open an active election (`/election/:id`).
2. Authenticate as a registered voter and initiate a vote (or follow the `addVote` code path in `ElectionsList.jsx` if no live election data is available).
3. Inspect the MetaMask transaction prompt and the corresponding contract call in the source code.

**Deliverables:**

- Document the call chain in order: **UI component → utility/helper → smart contract function**.
- List each argument passed to the on-chain vote function and explain where the frontend obtains each value (props, API response, `localStorage`, etc.).
- Identify one authorization or trust assumption in the frontend that is **not** enforced by the smart contract, and explain the on-chain impact.

---

### Problem 3 — Frontend & Backend API Integration Audit

**Objective:** Trace how the React client communicates with the Express/MongoDB API for voter-facing flows and evaluate the integration.

**Steps:**

1. With the backend and frontend running, open browser DevTools → **Network** tab.
2. Navigate to `/election` and record the request that populates the election list (or the empty-state message when no elections match).
3. Open an active election, select a candidate, and reach the voter login form at `/login`.
4. Submit the form with **invalid** credentials, then repeat with **valid** credentials (or inspect the equivalent `axios` calls in `ElectionsList.jsx`).
5. On the server, locate the matching route definitions and controller handlers.

**Deliverables:**

- Identify where the frontend configures the API base URL and where the backend mounts its auth router. State the full URL path used for voter login.
- Document the call chain for loading elections on `/election`: **UI component → HTTP method & path → route handler → database query/filter**.
- For voter login, list the request body fields, the HTTP status codes returned for success vs. failure, and what the frontend does with a successful response before proceeding to `addVote`.
- Identify one security or data-integrity issue in the FE/BE login flow (API layer only — not the smart contract), and explain its impact.

---

## Answer Key (For Evaluators)

<details>
<summary>Problem 1 — Expected findings</summary>

| Item | Expected answer |
|------|-----------------|
| Contract wiring | `client/src/utils/contractInstance.js` creates `ethers.providers.Web3Provider(window.ethereum)`, loads ABI from `./Voting.json`, and uses `contractAddress` exported in the same file. |
| Missing MetaMask | `TransactionContext.jsx` → `connectWallet()` shows alert: `"Please install MetaMask."` Contract calls via `contractInstance.js` will fail when `window.ethereum` is undefined. |
| Transaction signer | The connected MetaMask account (`signer.getAddress()`) becomes `msg.sender` for all contract writes. |

</details>

<details>
<summary>Problem 2 — Expected findings</summary>

| Item | Expected answer |
|------|-----------------|
| Call chain | `ElectionsList.jsx` (`addVote`) → `contractInstance.js` → `Voting.voteTo(...)` |
| Arguments | `voteTo(candidateAddress, organizerAddress, voterAddress, electionId)` — sourced from election/candidate API data, `localStorage` (`connected address`, `listsize`), and voter login response. |
| Security gap | The UI passes `_voter` as a function argument, but `voteTo` is `public` and never checks `msg.sender == _voter`. Any address can cast a vote on behalf of a registered voter who has not yet voted. |

</details>

<details>
<summary>Problem 3 — Expected findings</summary>

| Item | Expected answer |
|------|-----------------|
| API base URL | Frontend: `client/src/Data/Variables.jsx` exports `serverLink`. Backend: `server/index.js` mounts routes at `app.use("/api/auth", Auth)`. Full voter login URL: `POST http://localhost:5000/api/auth/login`. |
| Election list chain | `Election.jsx` → `GET serverLink + "voting/elections"` → `AuthRoute.js` → `elections.voting` in `AuthController.js` → `Election.find({ currentPhase: "voting" })`. Empty list shows "Election is not started yet!" |
| Voter login flow | Body: `{ username, password }`. Invalid user/password → `202` with plain-text message; success → `201` with full user document. On `201`, `ElectionsList.jsx` `handleSubmit` calls `addVote()`, using `location.state` (candidate/election fields) plus login response fields. |
| API security gap | Passwords stored and compared in plaintext (`findUser.password !== req.body.password`). No hashing, JWT, or session — successful login returns the entire user object (including password) to the client with no subsequent server-side authorization on vote actions. |

</details>

**Scoring:** 5 points per problem (15 points total). Award full marks when deliverables are accurate and clearly validated against the running frontend and Network tab.

---

## Project Structure

```
├── client/          React frontend (voter & admin UI)
├── server/          Express API and MongoDB models
├── hardhat/         Solidity contracts and deployment scripts
└── .env.sample      Environment variable template
```
