# คู่มือ Docker Lab: Next.js + TypeScript

คู่มือนี้อธิบาย Dockerfile, compose.yaml และ .env สำหรับโปรเจกต์นี้ ซึ่งมีไฟล์ทั้งสามอยู่ที่โฟลเดอร์หลักแล้ว ส่วน .dockerignore ด้านล่างเป็นตัวอย่างที่แนะนำ ตัวอย่างเป็นโหมดพัฒนา (development)

## ไฟล์ที่ใช้ในบทเรียน

| ไฟล์ | หน้าที่ |
| --- | --- |
| Dockerfile | บอกวิธีสร้าง image ของเว็บแอป |
| compose.yaml | กำหนด service, port และวิธีรัน container |
| .env | เก็บค่าที่ใช้ปรับการตั้งค่า เช่น port |
| .dockerignore | กันไฟล์ที่ไม่จำเป็นหรือข้อมูลลับออกจาก build context |

## 1. ตัวอย่าง Dockerfile

ตรวจดูไฟล์ Dockerfile ที่โฟลเดอร์หลักของโปรเจกต์ ตัวอย่างเนื้อหา:

~~~~dockerfile
FROM node:24-slim

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci

COPY . .

EXPOSE 3000

CMD ["npm", "run", "dev", "--", "--hostname", "0.0.0.0"]
~~~~

อธิบายทีละส่วน:

- FROM เลือก image พื้นฐานที่มี Node.js และ npm
- WORKDIR กำหนดโฟลเดอร์ทำงานภายใน container
- COPY package*.json และ npm ci ติดตั้ง dependencies จาก lockfile
- COPY . . นำ source code เข้า image
- EXPOSE บอกว่าแอปใช้ port 3000 ภายใน container
- CMD เปิด Next.js dev server และ bind กับ 0.0.0.0 เพื่อให้เข้าถึงจากเครื่อง host ได้

## 2. ตัวอย่าง compose.yaml

ไฟล์ compose.yaml อยู่ข้าง Dockerfile ตัวอย่างเนื้อหา:

~~~~yaml
services:
  web:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "${APP_PORT:-3000}:3000"
    environment:
      NODE_ENV: development
~~~~

รูปแบบ port คือ host:container ดังนั้นค่า 3000:3000 หมายถึงเปิด port 3000 บนเครื่อง host แล้วส่งต่อไปยัง port 3000 ใน container ชื่อ web เป็นชื่อ service ที่ใช้ดู log หรือสั่งงาน service นี้

Compose จะอ่าน .env เพื่อแทนค่า APP_PORT ใน compose.yaml โดยอัตโนมัติ ค่าใน .env สำหรับการแทนค่านี้ไม่ได้ถูกส่งเข้า container เว้นแต่จะระบุไว้ใน environment หรือ env_file ด้วย

## 3. ตัวอย่าง .env

ใช้ไฟล์ .env ที่โฟลเดอร์หลัก ข้าง compose.yaml โดยตัวอย่างนี้กำหนดค่า:

~~~~dotenv
APP_PORT=3000
~~~~

ถ้า port 3000 ถูกใช้อยู่ สามารถเปลี่ยนเป็น APP_PORT=3001 แล้วเปิดเว็บที่ http://localhost:3001 ได้ ส่วน port ภายใน container ยังเป็น 3000

ตัวอย่างนี้ไม่มีรหัสผ่านหรือ secret ห้ามเก็บ API key, password หรือ token จริงไว้ในตัวอย่างที่แชร์ ควรแยกค่าลับออกจากเอกสาร และไม่ส่งไฟล์ .env เข้า image

## 4. ตัวอย่าง .dockerignore ที่แนะนำ

ก่อน build ควรกัน dependencies, cache และไฟล์ environment ออกจาก build context สร้างไฟล์ .dockerignore ที่โฟลเดอร์หลัก:

~~~~text
node_modules
.next
.git
.env
.env.*
!.env.example
~~~~

โปรเจกต์นี้มี .gitignore ที่กันไฟล์ .env* ไว้แล้ว ส่วน .dockerignore เป็นไฟล์คนละหน้าที่ ใช้ควบคุมไฟล์ที่ Docker ส่งไปตอน build

## 5. คำสั่งสำหรับบทเรียน

เปิด Terminal ที่โฟลเดอร์โปรเจกต์ แล้วใช้คำสั่งเหล่านี้เมื่อพร้อมเริ่มใช้งาน Docker:

~~~~powershell
docker compose config
docker compose up --build
~~~~

คำสั่งแรกแสดง Compose configuration หลังแทนค่าจาก .env ส่วนคำสั่งที่สองสร้าง image แล้วเริ่ม service เมื่อเริ่มทำงานแล้ว เปิด http://localhost:3000 (หรือ port ที่กำหนดใน APP_PORT)

หยุด service ด้วย Ctrl+C แล้วสั่ง:

~~~~powershell
docker compose down
~~~~

แก้ไข source code แล้วต้องการให้ container ใช้ไฟล์ชุดใหม่ ให้สั่ง docker compose up --build อีกครั้ง ตัวอย่างนี้ยังไม่ mount source code เป็น volume เพื่อให้เห็นขั้นตอนการ build image อย่างชัดเจน

## ขอบเขตของตัวอย่าง

ตัวอย่างนี้ใช้ Next.js development server เพื่อการเรียนรู้และทดลองในเครื่อง ไม่ใช่การตั้งค่าสำหรับ production; Next.js แนะนำแนวทาง Docker แบบ standalone สำหรับสร้าง production image ที่เล็กลงและเตรียมเฉพาะไฟล์ที่จำเป็น

## 6. คู่มือ GitHub: สมัครบัญชี สร้าง repo และเชื่อมโปรเจกต์

ขั้นตอนนี้อธิบายการใช้ Git ในเครื่องร่วมกับ GitHub เป็น remote ปัจจุบันโปรเจกต์นี้ยังไม่มี Git repository และยังไม่ได้เชื่อมกับ GitHub; ทำตามขั้นตอนเมื่อพร้อมเริ่มใช้งาน

### 1) สมัครบัญชี GitHub

1. เปิด [github.com/signup](https://github.com/signup) แล้วทำตามขั้นตอนสมัคร
2. ยืนยันอีเมลให้เรียบร้อย เพราะ GitHub อาจจำกัดการสร้าง repository หากยังไม่ยืนยันอีเมล
3. เปิดใช้ two-factor authentication (2FA) เพื่อเพิ่มความปลอดภัยให้บัญชี

### 2) สร้าง repository บน GitHub ด้วยตนเอง

1. ลงชื่อเข้าใช้ GitHub แล้วเลือกปุ่ม **+** มุมขวาบน จากนั้นเลือก **New repository**
2. ตั้งชื่อ เช่น lab-docker-typescript-next-js
3. เลือก **Public** หากต้องการให้คนทั่วไปเห็น หรือ **Private** หากต้องการจำกัดการเข้าถึง
4. สำหรับ **Add a README file**, **Add .gitignore**, และ **Choose a license** ให้ปล่อยว่างไว้ เพื่อให้ repository บน GitHub เป็น repo เปล่าและหลีกเลี่ยงประวัติเริ่มต้นชนกับโปรเจกต์ในเครื่อง
5. เลือก **Create repository**
6. ในหน้า Quick setup ให้คัดลอก URL แบบ HTTPS ที่ลงท้ายด้วย .git เช่น https://github.com/YOUR-USERNAME/lab-docker-typescript-next-js.git

### 3) เตรียมโปรเจกต์ในเครื่อง

เปิด PowerShell ที่โฟลเดอร์โปรเจกต์:

~~~~powershell
Set-Location "$env:USERPROFILE/Desktop/lab-docker-typescript-next-js"
git --version
git init -b main
~~~~

ถ้า Git รุ่นเก่าไม่รองรับ git init -b ให้ใช้:

~~~~powershell
git init
git branch -M main
~~~~


ตั้งค่าชื่อและอีเมลที่จะแสดงใน commit สำหรับ repository นี้:

~~~~powershell
git config user.name "ชื่อที่ต้องการแสดง"
git config user.email "อีเมลที่ต้องการแสดงใน commit"
git config --local --list
~~~~

ถ้า repository เป็น Public อีเมลใน commit อาจมองเห็นได้ หากต้องการซ่อนอีเมลส่วนตัว ให้ใช้ GitHub noreply email ซึ่งดูได้จาก Settings > Emails ในบัญชี GitHub
### 4) บันทึกไฟล์และเชื่อมกับ repo ที่สร้างไว้

ก่อนบันทึก ให้ตรวจว่า .env หรือข้อมูลลับไม่อยู่ในรายการไฟล์:

~~~~powershell
git status
git check-ignore -v .env
git add .
git status
git diff --staged
git commit -m "chore: start Docker lab project"
~~~~

จากนั้นแทน YOUR-USERNAME และชื่อ repo ใน URL ด้วยค่าจริงที่คัดลอกจาก GitHub แล้วเชื่อม remote:

~~~~powershell
git remote add origin https://github.com/YOUR-USERNAME/lab-docker-typescript-next-js.git
git remote -v
git push -u origin main
~~~~

- origin เป็นชื่อเรียกสั้น ๆ ของ repository บน GitHub
- git remote -v ใช้ตรวจว่า URL ถูกต้อง
- git push -u origin main ส่ง commit แรกขึ้น GitHub และตั้งค่า upstream สำหรับ branch main
- หาก Git ขอให้ยืนยันตัวตน ให้ทำตามหน้าต่างลงชื่อเข้าใช้ของ Git Credential Manager หรือวิธี HTTPS authentication ที่ GitHub รองรับ ห้ามใส่รหัสผ่านหรือ token ลงใน URL หรือแชร์ในเอกสาร

โปรเจกต์นี้มี .gitignore ที่ยกเว้น .env*, node_modules และ .next ไว้แล้ว แต่ควรตรวจ git status ก่อนทุก commit และห้าม push API key, password หรือ token

### 5) การทำงานครั้งถัดไป

หลังแก้ไขไฟล์ ให้ตรวจรายการและบันทึก commit ใหม่ จากนั้น push ขึ้น GitHub:

~~~~powershell
git status
git add .
git diff --staged
git commit -m "docs: update Docker lab guide"
git push
~~~~

สร้าง branch สำหรับทดลองงานแยกจาก main ได้ด้วย:

~~~~powershell
git switch -c feature/git-practice
git switch main
~~~~

## เอกสารอ้างอิง

- [Next.js: Deploying with Docker](https://nextjs.org/docs/app/getting-started/deploying)
- [Next.js: output standalone](https://nextjs.org/docs/app/api-reference/config/next-config-js/output)
- [Docker Compose: services](https://docs.docker.com/reference/compose-file/services/)
- [Docker Compose: variable interpolation and .env](https://docs.docker.com/compose/how-tos/environment-variables/variable-interpolation/)
- [Docker Compose: getting started and .dockerignore](https://docs.docker.com/compose/gettingstarted/)
- [GitHub: creating an account](https://docs.github.com/en/account-and-profile/how-tos/account-management/creating-an-account-on-github)
- [GitHub: create a repository](https://docs.github.com/en/get-started/start-your-journey/creating-a-repository-for-your-project-on-github)
- [GitHub: add local code to GitHub](https://docs.github.com/en/migrations/importing-source-code/using-the-command-line-to-import-source-code/adding-locally-hosted-code-to-github)
- [GitHub: authentication](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/about-authentication-to-github)
- [GitHub: commit email address and privacy](https://docs.github.com/en/account-and-profile/how-tos/email-preferences/setting-your-commit-email-address)
- [Git: gitignore reference](https://git-scm.com/docs/gitignore)