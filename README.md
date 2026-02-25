<!-- Banner -->
<p align="center">
  <img src="https://capsule-render.vercel.app/api?type=waving&color=0:0d1117,50:161b22,100:30363d&height=220&section=header&text=Bizarro0x13&fontSize=70&fontColor=58a6ff&fontAlignY=35&desc=Smart%20Contract%20Security%20Researcher&descSize=20&descColor=8b949e&descAlignY=55&animation=fadeIn" width="100%"/>
</p>

<p align="center">
  <a href="https://twitter.com/Bizarro0x13"><img src="https://img.shields.io/badge/Twitter-1DA1F2?style=for-the-badge&logo=twitter&logoColor=white" alt="Twitter"/></a>
  <a href="https://code4rena.com/Bizarro0x13"><img src="https://img.shields.io/badge/Code4rena-000000?style=for-the-badge&logo=ethereum&logoColor=white" alt="Code4rena"/></a>
  <a href="https://audits.sherlock.xyz/watson/Bizarro"><img src="https://img.shields.io/badge/Sherlock-6C3EC1?style=for-the-badge&logo=ethereum&logoColor=white" alt="Sherlock"/></a>
  <a href="https://profiles.cyfrin.io/u/Bizarro0x13"><img src="https://img.shields.io/badge/CodeHawks-FF4500?style=for-the-badge&logo=ethereum&logoColor=white" alt="CodeHawks"/></a>
  <a href="https://cantina.xyz/u/Bizarro"><img src="https://img.shields.io/badge/Cantina-FF6B35?style=for-the-badge&logo=ethereum&logoColor=white" alt="Cantina"/></a>
  <a href="https://immunefi.com/profile/Bizarro"><img src="https://img.shields.io/badge/Immunefi-00C4B4?style=for-the-badge&logo=ethereum&logoColor=white" alt="Immunefi"/></a>
</p>

---

## 🛡️ About Me

> Independent smart contract security researcher focused on uncovering critical vulnerabilities in DeFi protocols, bridges, and novel on-chain architectures.

- 🔍 Specializing in **EVM-based** smart contract security
- 🧠 Deep expertise in **DeFi**, **Lending/Borrowing**, **DEXs**, **Bridges**, and **NFT protocols**
- 🛠️ Proficient in **Solidity**, **Foundry**, **Hardhat**, **Slither**, **Aderyn**, and manual review
- 📖 Constantly exploring new attack vectors and EVM internals

---

## 📊 Audit Stats at a Glance

<table align="center">
  <tr>
    <td align="center"><h3>🏆</h3></td>
    <td align="center"><h3>🐛</h3></td>
    <td align="center"><h3>🔴</h3></td>
    <td align="center"><h3>🟠</h3></td>
  </tr>
  <tr>
    <td align="center"><b>Contests</b></td>
    <td align="center"><b>Total Findings</b></td>
    <td align="center"><b>Highs</b></td>
    <td align="center"><b>Mediums</b></td>
  </tr>
  <tr>
    <td align="center"><code>20</code></td>
    <td align="center"><code>42</code></td>
    <td align="center"><code>17</code></td>
    <td align="center"><code>14</code></td>
  </tr>
</table>

---

## 🏅 Contest Highlights & Positions

| Contest | Platform | Findings | Rank |
|:--------|:---------|:---------|:-----|
| Infinifi-protocol | Cantina | 1H | #4 |
| primev-validator-registry | Cantina | 1H | #6 |
| octant-v2-core | Cantina | 2M | #7 |
| mystic-monorepo | Cantina | 7H, 5M | #9 |
| pike-tapio-monrepo | Cantina | 1M | #9 |
| IQ AI | Code4rena | 1M | #16 |
| telcoin-network | Cantina | 3 findings | #29 |
| Vechain — Stargate Hayabusa | Immunefi | 2 findings | #29 |
| succinct-network | Cantina | 1 finding | #31 |
| Liquidity Management | CodeHawks | 2 findings | #44 |
| Malda | Sherlock | 1 finding | #46 |
| DODO Cross-Chain DEX | Sherlock | 2 findings | #62 |

---

## 🔬 Notable Findings

<details>
<summary><b>🔴 High — Attacker can claim other user's refundAmount</b></summary>

- **Protocol:** DODO Cross-Chain DEX
- **Platform:** Sherlock
- **Impact:** An attacker can drain refund amounts belonging to other users by exploiting improper access control on the refund claim flow
- **Link:** [Report Link](#)
</details>

<details>
<summary><b>🔴 High — Finding not yet public</b></summary>

- **Protocol:** Infinifi-protocol
- **Platform:** Cantina
- **Impact:** Not yet disclosed
- **Link:** TBD
</details>

<details>
<summary><b>🔴 High — Finding not yet public</b></summary>

- **Protocol:** primev-validator-registry
- **Platform:** Cantina
- **Impact:** Not yet disclosed
- **Link:** TBD
</details>

<details>
<summary><b>🟠 Medium — onchain calculation of the amountInMax can cause attacker to sandwich the transaction</b></summary>

- **Protocol:** DODO Cross-Chain DEX
- **Platform:** Sherlock
- **Impact:** Attackers can profitably sandwich cross-chain swap transactions due to predictable on-chain `amountInMax` calculation, causing users to receive less than expected
- **Link:** [Report Link](#)
</details>

<details>
<summary><b>🟠 Medium — Deposits on long one leverage vault don't actually finalize the flow, leading to a Denial of Service (DoS)</b></summary>

- **Protocol:** Liquidity Management
- **Platform:** CodeHawks
- **Impact:** Users attempting to deposit into the long-one leverage vault encounter a permanent DoS, making the vault unusable
- **Link:** [Report Link](#)
</details>

<details>
<summary><b>🟠 Medium — Incorrect Token Price Validation in KeeperProxy</b></summary>

- **Protocol:** Liquidity Management
- **Platform:** CodeHawks
- **Impact:** Improper price validation in the KeeperProxy contract allows operations to proceed with stale or manipulated prices
- **Link:** [Report Link](#)
</details>

<details>
<summary><b>🟠 Medium — Incorrect Max Transfer Size Check in sendMsg Function</b></summary>

- **Protocol:** Malda
- **Platform:** Sherlock
- **Impact:** The max transfer size check in `sendMsg` is incorrectly implemented, allowing messages that exceed the intended size limit to be sent cross-chain
- **Link:** [Report Link](#)
</details>

<details>
<summary><b>🟠 Medium — Ineffective proposal threshold validation allows setting arbitrary high values</b></summary>

- **Protocol:** IQ AI
- **Platform:** Code4rena
- **Impact:** The proposal threshold can be set to arbitrarily high values due to flawed validation, effectively preventing any new governance proposals from being submitted
- **Link:** [Report Link](#)
</details>

---

## 📋 Full Audit Portfolio

| # | Protocol | Type | Platform | Date | Findings | Rank |
|:--|:---------|:-----|:---------|:-----|:---------|:-----|
| 1 | IQ AI | AI / DeFi | Code4rena | Jan '25 | 1M | #16 |
| 2 | Liquidity Management | Liquidity | CodeHawks | Feb '25 | 2 findings | #44 |
| 3 | Infinifi-protocol | Yield / DeFi | Cantina | Apr '25 | 1H | #4 |
| 4 | mystic-monorepo | Cross-chain / DeFi | Cantina | May '25 | 7H, 5M | #9 |
| 5 | primev-validator-registry | Infrastructure | Cantina | May '25 | 1H | #6 |
| 6 | DODO Cross-Chain DEX | DEX / Cross-chain | Sherlock | Jun '25 | 2 findings | #62 |
| 7 | telcoin-network | Cross-chain | Cantina | Jun '25 | 3 findings | #29 |
| 8 | DeBank | Portfolio / DeFi | Sherlock | Jul '25 | — | #107 |
| 9 | Malda | Lending | Sherlock | Jul '25 | 1 finding | #46 |
| 10 | pike-tapio-monrepo | Bridge / Stablecoin | Cantina | Jul '25 | 1M | #9 |
| 11 | succinct-network | ZK / Infrastructure | Cantina | Jul '25 | 1 finding | #31 |
| 12 | octant-v2-core | Governance / DeFi | Cantina | Sep '25 | 2M | #11 |
| 13 | Alchemix V3 | Yield / DeFi | Immunefi | Oct '25 | 3 findings | #121 |
| 14 | Vechain — Stargate Hayabusa | Bridge / Cross-chain | Immunefi | Nov '25 | 2 findings | #29 |

---

## 🧰 Tech Stack & Tools

<p align="center">
  <img src="https://img.shields.io/badge/Solidity-363636?style=for-the-badge&logo=solidity&logoColor=white"/>
  <img src="https://img.shields.io/badge/Foundry-000000?style=for-the-badge&logo=ethereum&logoColor=white"/>
  <img src="https://img.shields.io/badge/Hardhat-FFF100?style=for-the-badge&logo=ethereum&logoColor=black"/>
  <img src="https://img.shields.io/badge/Slither-2E86C1?style=for-the-badge&logo=python&logoColor=white"/>
  <img src="https://img.shields.io/badge/Aderyn-FF6B6B?style=for-the-badge&logo=rust&logoColor=white"/>
  <img src="https://img.shields.io/badge/Rust-000000?style=for-the-badge&logo=rust&logoColor=white"/>
  <img src="https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white"/>
  <img src="https://img.shields.io/badge/Echidna-5C2D91?style=for-the-badge&logo=ethereum&logoColor=white"/>
  <img src="https://img.shields.io/badge/Medusa-1B1F23?style=for-the-badge&logo=ethereum&logoColor=white"/>
</p>

---

## 📈 GitHub Activity

<p align="center">
  <img src="https://github-readme-stats.vercel.app/api?username=Bizarro0x13&show_icons=true&theme=github_dark&hide_border=true&bg_color=0d1117&title_color=58a6ff&icon_color=58a6ff&text_color=8b949e" height="170"/>
  <img src="https://github-readme-streak-stats.herokuapp.com/?user=Bizarro0x13&theme=github-dark-blue&hide_border=true&background=0d1117&ring=58a6ff&fire=58a6ff&currStreakLabel=58a6ff" height="170"/>
</p>

---

## 📬 Let's Connect

- 🐦 X: [Bizarro0x13](https://x.com/Bizarro0x13)
- 📧 Email: bizarro0x13@gmail.com
- 💬 Discord: `bizarro0x13`
- 🔗 Open for **private audits** and **security consultations**

---

<p align="center">
  <img src="https://capsule-render.vercel.app/api?type=waving&color=0:0d1117,50:161b22,100:30363d&height=120&section=footer" width="100%"/>
</p>

<p align="center">
  <i>"The best security researchers don't just find bugs — they think like attackers."</i>
</p>