const { chromium } = require("playwright");
const axios = require("axios");
require('dotenv').config();
const fs = require('fs');

fs.appendFileSync("log.txt", "RUN: " + new Date() + "\n");

const SHEET_URL= process.env.SHEET_URL;
const USERNAME = process.env.JOBBKK_USER;
const PASSWORD = process.env.JOBBKK_PASS;
const WEB = process.env.WEB;
const PAGE = process.env.PAGE;

// popup
async function handlePopup(page) {
  await page.locator('button:has-text("ยอมรับ"), button:has-text("Accept")').click().catch(() => {});
  await page.locator('button:has-text("ปิด")').click().catch(() => {});
  await page.locator('[class*="close"]').click().catch(() => {});
}

// parse date
function parseThaiDate(dateStr) {
  if (!dateStr) return null;

  const months = {
    "ม.ค.": 0, "ก.พ.": 1, "มี.ค.": 2, "เม.ย.": 3,
    "พ.ค.": 4, "มิ.ย.": 5, "ก.ค.": 6, "ส.ค.": 7,
    "ก.ย.": 8, "ต.ค.": 9, "พ.ย.": 10, "ธ.ค.": 11
  };

  const match = dateStr.match(/(\d{1,2})\s(.+)\s(\d{4})/);
  if (!match) return null;

  return new Date(
    parseInt(match[3]) - 543,
    months[match[2]],
    parseInt(match[1])
  );
}

// login
async function login(page) {
  console.log("login...");

  await page.goto(WEB);
  await page.waitForSelector('#username_emp');

  await page.fill('#username_emp', USERNAME);
  await page.fill('#password_emp', PASSWORD);

  await page.click('#sign_in_emp');
  await page.waitForTimeout(2000);

  await handlePopup(page);

  const confirmBtn = page.locator('button:has-text("ยืนยัน"), button:has-text("ตกลง")');

  if (await confirmBtn.isVisible({ timeout: 5000 })) {
    console.log("เจอ session เก่า → กดยืนยัน");

    await Promise.all([
      page.waitForLoadState('networkidle'),
      confirmBtn.click()
    ]);
  }

  await page.waitForTimeout(3000);

  if (page.url().includes("login")) {
    await page.goto(PAGE);
    await page.waitForTimeout(3000);
  }

  if (page.url().includes("login")) {
    throw new Error("login fail");
  }

  console.log("login ok");
}

(async () => {

  const browser = await chromium.launch({ headless: false });
  const context = await browser.newContext();
  const page = await context.newPage();

  try {
    await login(page);

    await page.goto("https://www.jobbkk.com/employer/icms");
    await page.waitForSelector('table tr');

    let selected = 0;
    const results = [];

    while (selected < 10) {

      await handlePopup(page);

      const rows = page.locator('table tr');
      const count = await rows.count();

      for (let i = 0; i < count; i++) {
        if (selected >= 10) break;

        const row = rows.nth(i);

        const linkEl = row.locator('a').first();
        if (await linkEl.count() === 0) continue;

        const name = await linkEl.innerText().catch(() => '');
        if (!name || name.includes('@') || name.split(' ').length < 2) continue;

        const text = await row.innerText().catch(() => null);
        if (!text) continue;

        // 🔹 split lines (ใช้ดึง position + exp)
        const lines = text.split('\n').map(t => t.trim()).filter(Boolean);

        const position = lines[1] || null;

        const scoreMatch = text.match(/(\d+)\s?%/);
        const score = scoreMatch ? parseInt(scoreMatch[1]) : 0;

        if (score < 80) continue;

        // 🔹 age + experience จาก list
        let age = null;
        let experience = null;

        const ageExpLine = lines.find(l => /\d+\s+(\d+|-)/.test(l));
        if (ageExpLine) {
          const parts = ageExpLine.split(/\s+/);
          age = parts[0] || null;
          experience = parts[1] && parts[1] !== '-' ? parts[1] + ' ปี' : null;
        }

        console.log(`เลือก: ${name} (${score}%)`);

        let newPage;
        try {
          [newPage] = await Promise.all([
            context.waitForEvent('page', { timeout: 5000 }),
            linkEl.click()
          ]);
        } catch {
          await linkEl.click();
          newPage = page;
        }

        await newPage.waitForLoadState('domcontentloaded');
        await handlePopup(newPage);

        const resumeText = await newPage.innerText('body');

        // 🔥 filter จังหวัด
        const isPathum =
          resumeText.includes("ปทุมธานี") ||
          resumeText.includes("ลำลูกกา");

        // 🔥 update
        const updateMatch = resumeText.match(/อัพเดตข้อมูลล่าสุด\s*:\s*(.+)/);
        const updateDate = updateMatch ? updateMatch[1].trim() : null;

        const parsedDate = parseThaiDate(updateDate);

        let isRecent = false;
        if (parsedDate) {
          const now = new Date();
          const diffMonth =
            (now.getFullYear() - parsedDate.getFullYear()) * 12 +
            (now.getMonth() - parsedDate.getMonth());

          isRecent = diffMonth <= 6;
        }

        if (!isPathum || !isRecent) {
          if (newPage !== page) await newPage.close();
          continue;
        }

        selected++;

        // 🔹 address (ปทุมธานี)
        const addressMatch = resumeText.match(/(.+(ปทุมธานี|ลำลูกกา))/);
        const address = addressMatch ? addressMatch[1].trim() : null;

        const phone = (resumeText.match(/0\d{8,9}/) || [null])[0];
        const email = (resumeText.match(/\S+@\S+\.\S+/) || [null])[0];

        let gender = null;
        const genderMatch = resumeText.match(/(หญิง|ชาย)/);
          if (genderMatch) {
              gender = genderMatch[1];
              }

        // 🔹 อายุจาก resume (override)
        const ageMatch2 = resumeText.match(/\((\d+)\s*ปี\)/);
        if (ageMatch2) age = ageMatch2[1];

        const nameParts = name.split(' ');
        const firstName = nameParts[0] || null;
        const lastName = nameParts.slice(1).join(' ') || null;

        const dateMatch = text.match(/\d{2}\/\d{2}\/\d{4}/);
        const applyDate = dateMatch ? dateMatch[0] : null;

        results.push({
          วันที่: applyDate,
          ตำแหน่ง: position,
          Resume_Score: score,
          Resume_Update: updateDate,
          ที่อยู่: address,
          ประสบการณ์: experience,
          เพศ: gender,
          อายุ: age,
          ชื่อ: firstName,
          นามสกุล: lastName,
          โทร: phone,
          อีเมล์: email,
          ลิงค์: newPage.url()
        });

        if (newPage !== page) await newPage.close();

        await page.waitForTimeout(800);
      }

      if (selected < 10) {
        const nextBtn = page.locator('text=ถัดไป');

        if (await nextBtn.isVisible()) {
          console.log("next page");

          await nextBtn.click();
          await page.waitForTimeout(2000);

        } else {
          console.log("no more page");
          break;
        }
      }
    }

    console.log("RESULT:", results);

    if (results.length > 0) {
      await axios.post(SHEET_URL, results);
      console.log("sent to sheet");
    }

  } catch (err) {
    console.log("ERROR:", err.message);
    fs.appendFileSync("log.txt", "ERROR: " + err.message + "\n");
  }

  await browser.close();

})();