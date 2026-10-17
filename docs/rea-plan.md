# REA (Reverse Engineer Anything): สรุปและแผนลองใช้

ที่มา: วิดีโอ "AI แกะโปรแกรมคนอื่นได้แล้ว? มารู้จัก REA GitHub Repo ที่กำลังเป็นไวรัล" ช่อง CodeKub
(https://www.youtube.com/watch?v=_NIfQl-WYOE) และ repo https://github.com/morluto/rea
(ตรวจจาก README ของ repo ณ เวอร์ชัน `rea-agents` 6.5.0)

## 1. สรุปสั้น

- REA คือ **MCP server + CLI** (แพ็กเกจ npm ชื่อ `rea-agents`, คำสั่ง `rea`) ที่ทำให้ AI coding agent
  เช่น Claude Code, Codex, Cursor, Gemini CLI เรียกใช้เครื่องมือ reverse engineering ได้
- แนวคิดคือ "เห็นฟีเจอร์ในแอปที่ชอบ → ให้ agent แกะดูว่าทำงานยังไง พร้อมหลักฐาน → เอาไปสร้างแบบเดียวกันในโปรเจกต์เรา"
- วิเคราะห์ในเครื่องเราเอง (local) ผลที่ได้จะแนบหลักฐาน (evidence) และข้อจำกัดของแต่ละข้อสรุปมาด้วย

## 2. กลุ่มเครื่องมือ (ตามวิดีโอ + ที่ repo ระบุเพิ่ม)

| กลุ่ม | ใช้กับ | ต้องมีอะไร |
| --- | --- | --- |
| Native binary | `.exe`, `.dll`, `.so`, Mach-O → ได้ pseudocode, assembly, strings, symbols, xrefs | Ghidra (ฟรี), Hopper หรือ IDA (มีลิขสิทธิ์) |
| JavaScript / Electron | โมดูล, imports, source map, route, IPC (renderer → preload → main) | Node.js + npm (Linux ต้องมี `ps` แบบ procps) |
| Browser / Runtime | โครงสร้างหน้าเว็บ, script, network request/response, screenshot | Chrome/Chromium |
| อื่น ๆ ที่ repo รองรับเพิ่ม | .NET, Android APK (JADX), Go/Rust binary, WASM, firmware, pcap/HAR, SQLite ฯลฯ | ดูตาราง "What you can analyze" ใน README |

## 3. สิ่งที่ต้องเตรียม (ทำบนเครื่องตัวเอง)

1. **Node.js** 22.x (>=22.19), 24.x (>=24.11) หรือ 26+ พร้อม npm (repo แนะนำ 24.18.0 ใน `.nvmrc`)
2. **Agent** ที่จะใช้ เช่น Claude Code
3. (ถ้าจะแกะ native binary) **Ghidra** + JDK แบบเต็ม เพราะเป็นตัวที่ฟรี
4. (ถ้าจะดูพฤติกรรมเว็บ) Chrome หรือ Chromium

## 4. ขั้นตอนติดตั้ง

```bash
# 1) ลงทะเบียน REA กับ agent (มันจะโชว์สิ่งที่จะแก้ไขให้กดยืนยันก่อน และสำรองไฟล์ config เดิมไว้ให้)
npx rea-agents setup                       # เลือก agent เอง
# หรือระบุเฉพาะ Claude Code
npx rea-agents setup --client claude_code

# 2) รีสตาร์ต agent

# 3) (ไม่บังคับ) ติดตั้ง CLI ไว้ใช้ประจำ
npm install --global rea-agents
rea --help
rea update                                  # อัปเดต (โปรเจกต์ออกเวอร์ชันใหม่บ่อย)
```

> หมายเหตุ: ถ้ารัน setup ให้ Claude Code มันจะเขียน `mcpServers.rea` ลงใน `~/.claude.json` และติดตั้ง skill
> ไว้ที่ `~/.claude/skills` ซึ่งจะมีผล**กับทุกโปรเจกต์** ถ้าอยากแยกไว้ใช้เฉพาะ ให้ใช้ profile แยกแบบนี้:
>
> ```bash
> CLAUDE_CONFIG_DIR=~/.claude-rea npx rea-agents setup --client claude_code
> CLAUDE_CONFIG_DIR=~/.claude-rea claude
> ```

## 5. แผนลองใช้ (เริ่มจากง่ายไปยาก ใช้เป้าหมายที่เรามีสิทธิ์วิเคราะห์)

### ขั้นที่ 1: JavaScript/Electron แบบ static (ไม่ต้องใช้ Ghidra)
- เป้าหมาย: แอป Electron ที่เป็น open source หรือแอปที่เราเขียนเอง (เช่น ลอง build `electron-quick-start` ขึ้นมาเป็นตัวอย่าง)
- คำสั่ง:
  ```bash
  npx -y rea-agents@latest analyze-javascript-application /absolute/path/to/app --json
  ```
- ดูผล: modules, imports, ขอบเขต Electron (main/preload/renderer) และ IPC

### ขั้นที่ 2: ให้ agent สืบฟีเจอร์ผ่าน MCP
- ตัวอย่าง prompt:
  ```text
  Understand how <ฟีเจอร์> works in <แอป>, show me the evidence, and build a similar feature for my project.
  ```
- ลองเลียนแบบ showcase "Notion clipboard bridge" คือไล่ตาม API ฝั่ง renderer → preload → IPC → main process

### ขั้นที่ 3: Native binary ด้วย Ghidra
- เป้าหมาย: โปรแกรม C เล็ก ๆ ที่เราคอมไพล์เอง แล้วเทียบ pseudocode ที่ได้กับ source จริง จะได้รู้ว่าผลแม่นแค่ไหน
- ตั้งค่า provider ตาม `docs/installation.md#ghidra` และ `docs/cli.md#choose-a-provider` ใน repo
- ถ้า binary ใหญ่ ให้เพิ่มเวลารอเริ่มทำงานด้วย `REA_GHIDRA_STARTUP_TIMEOUT_MS`

### ขั้นที่ 4: Browser / Runtime
- เป้าหมาย: เว็บของเราเอง หรือเว็บ localhost แล้วดู network request และ script ที่ทำงานจริง

## 6. ข้อควรระวัง (จาก DISCLAIMER.md)

- ใช้ได้เพื่อการเรียนรู้, งานวิจัยด้านความปลอดภัย, interoperability หรือกับซอฟต์แวร์ที่เราเป็นเจ้าของหรือได้รับอนุญาตเท่านั้น
- **ห้าม**ใช้ทำสิ่งเหล่านี้: bypass DRM, license check หรือ anti-cheat, ทำโปรแกรมโกงเกม, ข้าม auth หรือ access control, คัดลอกโค้ดหรือ asset ที่มีลิขสิทธิ์
- ผลที่ได้จาก decompile และจาก AI อาจไม่ครบหรือผิด ต้องตรวจหลักฐานเองก่อนเชื่อ
- เวลาแกะฟีเจอร์ไปทำใหม่ ให้เขียนขึ้นเองจากความเข้าใจพฤติกรรม ห้ามก๊อปโค้ดที่ decompile ได้ไปใช้ตรง ๆ
- REA ประมวลผลในเครื่อง แต่ผลลัพธ์จะถูกส่งให้ AI model ดังนั้นอย่าใช้กับข้อมูลลับ
- รันแอปที่ไม่น่าไว้ใจใน VM หรือ sandbox เสมอ

## 7. Checklist

- [ ] ติดตั้ง Node.js 22.19+/24.11+
- [ ] `npx rea-agents setup --client claude_code` แล้วรีสตาร์ต agent
- [ ] ทำขั้นที่ 1: วิเคราะห์ Electron app ตัวอย่าง
- [ ] ทำขั้นที่ 2: ให้ agent อธิบายฟีเจอร์หนึ่งอย่างพร้อมหลักฐาน
- [ ] ติดตั้ง Ghidra + JDK แล้วทำขั้นที่ 3
- [ ] ทำขั้นที่ 4: วิเคราะห์ runtime ของเว็บ
