# Lab 13 — Deploy One Working Backend

**Compose overlays · Local Nginx Proxy Manager · Ubuntu VM · Railway**

**Dr. Pisit Praiwattana, your mentor in 2026**

Website: [Lab 13](https://pisit-devop.com/itcs258/Lab13/)

Use the **same gacha workshop** throughout required experiments **01–04**. Run one environment at a time. Other backend examples are optional extension practice, not another required project.

ใช้ **gacha workshop ชุดเดิม** ตลอดงานหลัก **01–04** รัน environment เดียวในแต่ละช่วง ตัวอย่างแบ็กเอนด์อื่นเป็นการฝึกเสริม ไม่ใช่โปรเจกต์บังคับเพิ่ม

## 01 — Select configuration and test direct access / เลือกค่าตั้งและทดสอบเข้าตรง

1. Download and extract `gacha-workshop.zip`. Copy `.env.example` to `.env`; set your student ID and local database values.
2. Explain which settings belong in `compose.yml` and which are added by `compose.local.yml`. Use explicit `-f` arguments so the automatically loaded `compose.override.yml` is not mixed into this selection.
3. Validate, build and start the local stack:

```text
docker compose --env-file .env -f compose.yml -f compose.local.yml config --quiet
docker compose --env-file .env -f compose.yml -f compose.local.yml up -d --build --wait
docker compose --env-file .env -f compose.yml -f compose.local.yml ps
```

4. Open `http://localhost:8085/test`. Use your own ID: health → account → recharge 1000 coins → exchange 1000 coins → draw → history. Record the actual draw/history ID, result and creation time.
5. Preview the staging and production configuration with `config`; do not start three stacks. Explain why port lists can merge and why chaining the local overlay before production does not automatically remove local published ports.

```text
docker compose --env-file .env -f compose.yml -f compose.staging.yml config --quiet
docker compose --env-file .env -f compose.yml -f compose.production.yml -f compose.proxy.yml config --services
```

Keep full resolved configuration private if it contains secrets. Your report should use sanitized snippets. The required environment file is `.env`; separate staging/production secret files are optional later practice.

แตกไฟล์ ตั้ง `.env` ตรวจค่าตั้ง แล้วเริ่ม local stack แสดงผล health และธุรกรรมจริงของตนเอง Preview staging/production ด้วย config โดยไม่เปิดสาม stack อธิบายการ merge ports และปิดข้อมูลลับในภาพรายงาน

## 02 — Route through local NPM / ส่งคำขอผ่าน NPM ในเครื่อง

Stop the local containers **without `-v`**, then start the same project using production + proxy overlays. Keep `STUDENT_ID` unchanged so the named database volume is reused on this same Docker host.

```text
docker compose --env-file .env -f compose.yml -f compose.local.yml down
docker compose --env-file .env -f compose.yml -f compose.production.yml -f compose.proxy.yml up -d --build --wait
docker compose --env-file .env -f compose.yml -f compose.production.yml -f compose.proxy.yml ps
```

- Open NPM admin at `http://127.0.0.1:81`; complete the initial administrator setup shown by your version.
- Add `127.0.0.1 gacha.test` to your laptop's hosts file, preserving existing entries.
- Create a Proxy Host for `gacha.test`, scheme `http`, forward hostname **`app`**, port **`3000`**. Use local HTTP for this `.test` name.
- Add a **Custom Location** `/backend/`, also forwarding to `http`, `app`, `3000`. In that location's gear/advanced field put only:

```nginx
rewrite ^/backend/(.*)$ /$1 break;
```

- Save and open **`http://gacha.test/backend/test`**. Check that health and all logic buttons work. The proxy sends `/backend/test` to the app as `/test`; its API buttons must also pass through `/backend/`.
- Retrieve the history row saved in experiment 01 without drawing a replacement. Confirm the database volume identity is unchanged.
- Draw the path: laptop browser → NPM container → `app:3000` → `db:5432`. Explain why **`localhost` inside NPM means NPM itself**, not the separate app container.

NPM creates the surrounding location/proxy configuration. Do not paste another location block or duplicate `proxy_pass` into its custom-location field. The app and NPM share this project's default Docker network; the app and database need no public host ports.

หยุด local โดยไม่ใส่ `-v` แล้วเลือก production + proxy ด้วย student ID เดิม ตั้ง hosts และ Proxy Host ให้ใช้ **app:3000** เพิ่ม Custom Location และ rewrite ทดสอบ **/backend/test** พร้อมปุ่ม แล้วอ่าน history เดิม อธิบาย localhost ภายใน NPM และ network ร่วมกัน

**Production note:** NPM is used to teach routing quickly. This course recommends reviewed, version-controlled Nginx configuration or a managed platform for real production, together with appropriate application authentication and operational controls. This is a course deployment guideline, not a claim that every NPM installation is unsafe.

## 03 — Operate an Ubuntu VM / ดูแล Ubuntu VM

Use an Ubuntu VM with a normal sudo-capable account and Docker Engine. This is the VPS model; Docker-in-Docker is not required. A classroom starting allocation is 2 vCPU, 4 GB RAM and 25 GB free disk, adjusted to your machine's capacity.

1. Identify the guest OS, address, disk and Docker service. Explain bridged versus NAT access. A NAT VM is not automatically a public server.
2. Use basic operations taught in class: `ssh`, `pwd`, `ls`, `cd`, `mkdir`, `cp`, `nano`, `sudo apt`, `systemctl`, `docker compose ps` and `logs`.
3. Transfer the same sanitized project into `~/apps/gacha-workshop`. Create the VM's own `.env`. Do not copy laptop database files as a running-container migration.
4. Start the stack in Ubuntu:

```text
sudo docker compose --env-file .env -f compose.yml -f compose.production.yml -f compose.proxy.yml up -d --build --wait
sudo docker compose --env-file .env -f compose.yml -f compose.production.yml -f compose.proxy.yml ps
sudo docker compose --env-file .env -f compose.yml -f compose.production.yml -f compose.proxy.yml logs --tail 30 app db npm
```

5. Keep NPM admin loopback-only. From the laptop, use an SSH tunnel, replacing the address with your actual VM address:

```text
ssh -L 8181:127.0.0.1:81 student@192.168.1.50
```

Open `http://127.0.0.1:8181`, configure the VM's NPM host, and point the laptop's `gacha.test` entry to the VM address when bridged networking is used. NAT users must use their actual host-to-guest forwarded HTTP/SSH ports.

6. Test `/backend/test` through the VM and show the network path. Create a VM record and demonstrate it remains after replacing the app container. VM storage is a new location; laptop records are not expected to appear automatically.
7. Explain the public-VPS extension: real DNS, provider firewall, HTTP/HTTPS, SSH access and a valid certificate. Only perform these changes on your own assigned server. Preserve the actual SSH port before enabling UFW, and remember Docker-published ports can bypass UFW filtering. Keep DB/app/admin ports private by design.

ใช้ Ubuntu VM จริงสำหรับฝึก VPS แสดงคำสั่งพื้นฐาน สถานะ Docker และ IP guest ย้ายซอร์สเดิม สร้าง `.env` ใหม่ ใช้ SSH tunnel เข้า admin NPM ทดสอบ /backend/test ผ่าน VM และพิสูจน์ข้อมูลของ VM หลังเปลี่ยน app container อธิบาย DNS/HTTPS/firewall สำหรับ VPS สาธารณะตามที่เรียน

## 04 — Deploy the same app on Railway / Deploy แอปเดิมบน Railway

Use the account/project arrangement approved by the instructor. A hosted platform requires its own service configuration; local Compose DNS names and volumes do not migrate automatically.

1. Push the sanitized gacha source to your own accessible Git repository.
2. Create an app service and PostgreSQL service in one Railway project/environment. For the course repository structure, select app Root Directory `/workshop/gacha`; for a repository containing only the downloaded app, select `/`.
3. Confirm the build uses the app's Dockerfile. Reference the actual PostgreSQL service's **private** `DATABASE_URL` in the app; do not enter Compose hostname `db` as a guessed hosted endpoint.
4. The app listens on `0.0.0.0` and uses `PORT`. Its idempotent startup creates missing base tables; later schema changes still require migrations.
5. Configure `/health`, generate the app's HTTPS domain, then test **`https://YOUR-APP-DOMAIN/test`**. This direct platform domain uses `/test`; `/backend/` belonged to your earlier Nginx mapping.
6. Recharge virtual coins, exchange, draw, and save an actual online history ID/timestamp. Redeploy **only the app**, then retrieve that same history row without creating a replacement.
7. Verify persistent database storage and identify the backup/recovery choices for the selected service. Explain how any required local data would be migrated separately.

Current Railway supports partial Compose import as an optional convenience. The core exercise explicitly creates and checks the two services so students can explain the mapping. Never assume imported ports, volumes, secrets or local records are automatically correct.

สร้าง app และ PostgreSQL บน Railway ใช้ Dockerfile ของแอปและตัวแปรเชื่อมต่อฐานข้อมูลจริง ทดสอบ HTTPS /test และ /health เก็บแถวออนไลน์หนึ่งแถว แล้วอ่านแถวเดิมหลัง redeploy เฉพาะแอป ตรวจ persistent storage และอธิบายว่าการย้ายข้อมูลในเครื่องเป็นอีกขั้นตอนหนึ่ง

## 05 — Optional: Render backend + Vercel visual tester / เสริม: แบ็กเอนด์บน Render + visual tester บน Vercel

This extension adds a second hosting pattern: a **static page** on Vercel calling an **API container** on Render (or your Railway app from 04). It is optional and does not change the required 01–04 evidence.

The tester lives in `gacha-workshop/visual-tester/` and also ships inside the app at **`/visual/`**. It is a p5.js request-path diagram you can play: each click sends a real request and animates page → entry → app → database.

1. **Local first, three ways.** Open `http://localhost:8085/visual/` (same origin, no CORS). Then serve the tester separately, either with npm or without it:

```text
cd visual-tester
npm start                       # with npm; no packages to install
python3 -m http.server 5500     # without npm (or: node serve.js)
```

   Set `CORS_ORIGINS=http://localhost:5500` in `.env`, recreate the app (`... up -d --wait app`), then Connect from `http://localhost:5500/`. Capture one **CORS ✕** before setting the variable and one **200** after it.
2. **Render backend.** Create Render Postgres (region close to you, version 18, Free) and copy its **Internal Database URL**. Create a Web Service from your GitHub repository: Language **Docker**, Root Directory `workshop/gacha` (or empty if the app is your repo root), Health Check Path `/health`, environment `DATABASE_URL=<internal URL>`. Test `https://YOUR-APP.onrender.com/test`.
3. **Vercel tester.** Import the same repository into Vercel with Root Directory `workshop/gacha/visual-tester`, Framework Preset **Other** and no build command. Add the exact production origin (for example `https://your-tester.vercel.app`) to the backend's `CORS_ORIGINS` and redeploy the backend.
4. Open `https://your-tester.vercel.app/?api=https://YOUR-APP.onrender.com` and run Health → Wallet → Recharge → Exchange → Draw → History. Verify the preflight with `curl -i -X OPTIONS …` (see slide *CORS · the API decides*).
5. Record a **cold start**: after 15+ idle minutes the first free Render request may take about a minute (the tester shows *waking up?*), and the second request is fast. Free Render Postgres expires 30 days after creation; do not keep evidence only there.
6. Explain why the Vercel tester cannot reach `http://VM-IP/backend` (mixed content) and why it may ask for local-network permission for `localhost`.

The optional `render.yaml` Blueprint declares the same two services. Use it only after the dashboard path works.

ส่วนเสริมนี้แยก **หน้าเว็บ static** บน Vercel ออกจาก **API container** บน Render (หรือ Railway จากข้อ 04) ไม่บังคับ และไม่เปลี่ยนหลักฐานที่ต้องส่งของ 01–04 ทดลอง tester ในเครื่องสามแบบ (/visual/, npm start, python3 -m http.server) ตั้ง CORS_ORIGINS ให้ตรง origin ของหน้าเว็บ แล้ว deploy แบ็กเอนด์บน Render ด้วย Dockerfile เดิมและ Internal Database URL จากนั้น host tester บน Vercel บันทึกผล CORS ✕ ก่อนตั้งค่า, 200 หลังตั้งค่า และ cold start ของบริการฟรี

## 06 — Optional: another backend / เสริม: แบ็กเอนด์อื่น

Adapt another backend from Lab 11/12 to this same deployment pattern. This is optional practice and does not add another required deployment or report.

นำแบ็กเอนด์อื่นจาก Lab 11/12 มาปรับใช้ได้เป็นแบบฝึกเสริม ไม่ต้องเพิ่มอีกโปรเจกต์หรือรายงานบังคับ

## In-room check and report / ตรวจในห้องและรายงาน

- Complete required parts **01–04** and show your actual running experiment to the **LA in the room before leaving**.
- Use your own screen/CLI/API results. Include selected configuration, a request-path diagram for each stage, actual health and logic-button results, and persistence evidence.
- Include a problem, diagnosis and fix (or a preventive check), plus your repository URL and sanitized relevant snippets.
- Complete the web self-quiz after the workshop and review its feedback. It is a practice module; Moodle assessment instructions remain separate.
- Submit **one `student_ID_lab13.PDF`** to **MyCourse**, replacing `student_ID` with your actual ID. Follow the opening/deadline shown there. Use the report template and check PDF readability after upload.

ทำ 01–04 แล้วแสดงงานจริงให้ LA ตรวจในห้องก่อนออก เก็บภาพหน้าจอของตนเอง พร้อมค่าตั้ง แผนภาพ คำสั่ง ผล API และหลักฐาน persistence ทำ self-quiz เพื่อทบทวน แล้วส่ง **student_ID_lab13.PDF ไฟล์เดียวใน MyCourse** ตามกำหนด ใช้รหัสนักศึกษาจริงและตรวจว่าไฟล์อ่านได้
