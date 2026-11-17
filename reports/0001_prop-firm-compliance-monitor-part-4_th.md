# รายงานวิจัย 0001
## Building a Prop Firm Compliance Monitor in MQL5 (Part 4): Economic Calendar and Account Exposure Monitoring

- **ผู้เขียน:** Chacha Ian Maroa
- **วันที่เผยแพร่:** 16 พฤศจิกายน 2026
- **วันที่ทำ Research:** 17 พฤศจิกายน 2026
- **บทความต้นฉบับ:** https://www.mql5.com/en/articles/24399
- **หมวดหมู่:** Trading Infrastructure / Risk Controls / MQL5 Engineering
- **ประเภทงาน:** Technical Design & Code-Architecture Review (ไม่ใช่งานทดสอบผลตอบแทนของกลยุทธ์)

---

## บทสรุปสำหรับผู้วิจัย

บทความตอนที่ 4 เพิ่มระบบติดตามข่าวเศรษฐกิจระดับทั้งบัญชี (Account-wide Economic News Monitoring) ให้กับ EA สำหรับตรวจสอบข้อกำหนดของ Prop Firm แนวทางหลักคืออ่าน Position ที่เปิดอยู่ทั้งหมด แปลง Symbol ที่รองรับให้เป็นชุดสกุลเงิน (Currency Set) แบบไม่ซ้ำ จากนั้นเรียกใช้ Economic Calendar ของ MetaTrader 5 เพื่อค้นหา Event ที่เกี่ยวข้องกับสกุลเงินเหล่านั้น เลือก Event ที่กำลังอยู่ในช่วงจำกัดการเทรดหรือกำลังจะเกิดขึ้น แล้วเผยแพร่สถานะให้ส่วนอื่นของ EA นำไปใช้

จุดแข็งที่สุดของการออกแบบคือ **การแยกสถานะข้อมูลและผลลัพธ์ให้ชัดเจน (Fault-aware State Modelling)** เช่น “ค้นหาแล้วไม่พบ Event” ต้องไม่ถูกตีความเหมือน “ระบบดึงข้อมูลไม่ได้” และ “ไม่พบ Exposure ที่รองรับ” ก็ไม่ได้หมายความว่าบัญชีไม่มีความเสี่ยง

**ข้อสรุป:** เป็นรูปแบบการออกแบบ Monitoring และ Risk Control ที่มีประโยชน์สำหรับต่อยอด แต่ยังไม่ใช่ Compliance Engine ที่พร้อมใช้งานกับทุก Broker หรือทุก Prop Firm และไม่ได้พิสูจน์ว่าช่วยสร้าง Alpha หรือเพิ่มผลตอบแทน

ข้อจำกัดสำคัญคือบทความนี้เป็นส่วนติดตามสถานะ ไม่ได้ปฏิเสธคำสั่งซื้อขายหรือปิด Position โดยอัตโนมัติ อีกทั้งไม่มีผลทดสอบ PnL, Drawdown Reduction, False-positive/False-negative Rate, API Latency หรือสถิติความน่าเชื่อถือของระบบ

## 1. คำถามวิจัยและขอบเขต

**คำถามหลัก:** เราจะออกแบบ EA บน MQL5 ให้ตรวจจับ Economic Calendar Event ที่เกี่ยวข้องกับ Position ทั้งบัญชี สร้างสถานะจำกัดการซื้อขายก่อนและหลังข่าว และไม่สับสนระหว่างข้อมูลว่างกับข้อมูลที่ดึงไม่สำเร็จได้อย่างไร?

บทความนี้เป็นส่วนที่ 4 ของชุด Prop Firm Guard โดยอาศัยองค์ประกอบพื้นฐานจากตอนก่อนหน้า เช่น Persistence, Session Monitoring, Drawdown และ Profit Target แล้วเพิ่มชั้น News Monitoring เข้าไป

ขอบเขตที่ต้องแยกให้ชัด:
- ระบบตรวจสอบ **Position ที่เปิดอยู่ในปัจจุบัน** ไม่ใช่ Exposure ของคำสั่งใหม่ที่กำลังจะส่ง
- การแปลง Symbol รองรับ Forex ทั่วไป และมีกรณีพิเศษ XAU/USD แบบเจาะจง
- การกรองข่าวใช้ระดับความสำคัญ (Importance) และเวลา Event ไม่ได้วิเคราะห์ Surprise, Consensus Deviation, Market Impact หรือความอ่อนไหวของแต่ละ Instrument
- บทความมีรายการทดสอบด้วยตนเอง แต่ไม่ได้แสดง Automated Unit/Integration Test Suite ที่ครอบคลุม

## 2. กลไกการทำงานของระบบ

### 2.1 ลำดับการประมวลผล

1. ตรวจสอบ Configuration เช่น ค่า Pre/Post News Window ต้องไม่ติดลบ และ Importance ต้องเป็น Enum ที่รองรับ
2. อ่านเวลาจาก Trade Server ให้ได้ค่าที่ใช้งานได้
3. วนอ่าน Position ที่เปิดอยู่ทั้งหมดในบัญชี
4. อ่านข้อมูล Symbol จาก Broker เช่น `SYMBOL_CURRENCY_BASE`, `SYMBOL_CURRENCY_PROFIT` และ `SYMBOL_TRADE_CALC_MODE`
5. แปลง Position ที่รองรับเป็น Base Currency และ Profit Currency; ใช้กฎพิเศษ XAU/USD ตามที่บทความกำหนด
6. รวม Currency เป็น Set และตัดค่าที่ซ้ำกัน
7. สร้างช่วงเวลาค้นหา Calendar ให้ครอบคลุมทั้ง Event ที่เพิ่งเกิดไปแล้วและ Event ในอนาคต
8. เรียก `CalendarValueHistory()` แยกตาม Currency จากนั้นอ่าน Metadata ด้วย `CalendarEventById()` และกรองตาม Importance กับเวลา
9. จัดอันดับ Event โดยให้ Event ที่กำลัง Restricted มีลำดับสูงกว่า Event ในอนาคต หากมีหลาย Event ที่ Active ให้เลือก Event ที่ใกล้เวลาปัจจุบันที่สุดในเชิงค่าสัมบูรณ์ ส่วนกรณีไม่มี Event Active ให้เลือก Event ในอนาคตที่ใกล้ที่สุด
10. สร้าง Restriction Window จากเวลา Event และอัปเดต State/Cache
11. สแกน Exposure ใหม่ทุก Timer Cycle แต่จำกัดการเรียก Calendar ด้วย Cache โดย Refresh เมื่อ Exposure เปลี่ยน, Cache หมดอายุ/ไม่ถูกต้อง, เวลาเดินย้อนกลับ หรือ Restriction ที่เลือกหมดอายุ

### 2.2 การแปลง Exposure จาก Symbol

ตัวอย่าง Mapping สำหรับ Forex:

| Position | Currency Set |
|---|---|
| EURUSD | EUR, USD |
| GBPJPY | GBP, JPY |
| EURUSD + GBPUSD | EUR, USD, GBP (ตัด USD ที่ซ้ำกัน) |

กรณีทองคำ บทความมีเงื่อนไขเฉพาะเมื่อ Metadata จาก Broker ระบุ Base Currency เป็น `XAU` และ Profit Currency เป็น `USD` จึงเพิ่ม USD เข้า Exposure Set เท่านั้น นี่เป็นกฎเฉพาะของ Implementation ไม่ใช่ Mapping มาตรฐานที่รับประกันว่าจะใช้ได้กับ Gold/CFD ทุก Broker

Symbol ที่มี Calculation Mode ไม่รองรับจะถูกข้าม ส่วนกรณีอ่าน Metadata ที่จำเป็นไม่ได้จะทำให้ผลสแกน Exposure ไม่สมบูรณ์ ระบบยังเก็บ Currency ที่อ่านได้จาก Position อื่นไว้เพื่อใช้วิเคราะห์ แต่ไม่ควรประกาศว่าผลรวมสมบูรณ์

### 2.3 State Machine

| State | ความหมาย |
|---|---|
| `NEWS_DISABLED` | ปิดการทำงานของ News Monitoring |
| `NEWS_NO_EXPOSURE` | สแกนครบแล้ว แต่ไม่พบ Holding Currency ที่ Mapping รองรับ |
| `NEWS_NO_EVENT` | ทราบ Exposure และค้นหา Calendar สำเร็จทุกส่วน แต่ไม่พบ Event ที่ผ่านเงื่อนไข |
| `NEWS_SAFE` | พบ Event ที่เกี่ยวข้อง แต่เวลาปัจจุบันอยู่นอก Restriction Window |
| `NEWS_RESTRICTED` | เวลาปัจจุบันอยู่ภายใน Restriction Window |
| `NEWS_UNAVAILABLE` | ไม่สามารถยืนยัน Exposure, Server Time, Calendar Data หรือ Event Metadata ที่จำเป็นได้ |

ผลลัพธ์ระดับ Search ควรแยก Error (`-1`), Valid Empty (`0`) และ Found (`1`) เช่นเดียวกับผลลัพธ์ระดับ Symbol Mapping ที่ควรแยก Error, Unsupported และ Supported

การแยกสถานะเหล่านี้เป็นแกนสำคัญของระบบ เพราะป้องกันไม่ให้ความผิดพลาดด้าน Data Retrieval ถูกตีความเป็นการไม่มีความเสี่ยง

## 3. คณิตศาสตร์ของช่วงเวลาจำกัดการซื้อขาย

กำหนดให้:
- (t_e) = เวลา Event ตาม Trade Server
- (t) = เวลาปัจจุบันตาม Trade Server
- (B) = จำนวนนาทีก่อน Event ที่ต้องจำกัดการซื้อขาย
- (A) = จำนวนนาทีหลัง Event ที่ต้องจำกัดการซื้อขาย

ช่วงเวลาจำกัดคือ

[
[t_e - 60B,; t_e + 60A]
]

สถานะ Restricted เมื่อ

[
t_e - 60B leq t leq t_e + 60A
]

หรือกำหนด (Delta = t_e - t) จะได้ว่า Restricted เมื่อ

[
-60A leq Delta leq 60B
]

**ตัวอย่าง:** กำหนด (B=A=10) นาที และ Event เกิดเวลา 14:00 ระบบต้องจำกัดช่วง 13:50 ถึง 14:10 (รวมขอบเขตทั้งสองด้าน) เวลา 13:55 จะยัง Restricted และเวลา 14:05 ก็ยัง Restricted ส่วน 14:11 จะพ้นช่วงของ Event นี้แล้ว

ช่วงค้นหา Calendar ต้องย้อนกลับไปครอบคลุม Post-event Window ด้วย ไม่ใช่ค้นหาเฉพาะ Event ในอนาคต มิฉะนั้น Event ที่เพิ่งประกาศไปไม่กี่นาทีอาจหายจากผลค้นหา ทั้งที่ Post-news Restriction ยังมีผลอยู่

**Clock Domain:** ฟังก์ชัน Economic Calendar ของ MQL5 ใช้เวลา Trade Server ไม่ใช่ Local Time ของเครื่องผู้ใช้ การเปรียบเทียบกับ `TimeTradeServer()` จึงเป็นแนวทางที่ถูกต้อง ส่วน Log และระบบภายนอกควรระบุ Timezone ให้ชัดเจน

## 4. การทบทวน Implementation และสถาปัตยกรรม MQL5

### 4.1 จุดแข็ง

- **ใช้ Broker Metadata แทนการแยกชื่อ Symbol:** การ Parse ชื่อ Symbol ด้วย String อาจพังเมื่อ Broker ใช้ Prefix/Suffix การอ่าน Symbol Properties เป็นแนวทางพื้นฐานที่เหมาะสมกว่า
- **แยก Unsupported ออกจาก Error:** Instrument ที่ไม่รองรับไม่ใช่เหตุการณ์เดียวกับการอ่านข้อมูลไม่สำเร็จ
- **รักษาข้อมูล Partial Scan:** Currency ที่อ่านได้จาก Position อื่นยังคงเก็บไว้เพื่อวิเคราะห์ แต่ State สุดท้ายยังต้องสะท้อนว่า Exposure ไม่สมบูรณ์
- **แยก Valid Empty จาก API Failure:** `CalendarValueHistory()` คืนจำนวนข้อมูลเมื่อสำเร็จ และคืน `-1` เมื่อเกิดข้อผิดพลาด ค่า 0 จึงเป็นผลลัพธ์ว่างที่ถูกต้อง ไม่ใช่ Error
- **จัดอันดับ Event แบบ Deterministic:** ใช้กฎเลือก Event ที่ชัดเจน ช่วยให้ Input ชุดเดิมให้ผลลัพธ์เดิม
- **Invalidate Cache เมื่อ Exposure เปลี่ยน:** ป้องกันการใช้ Event เก่าที่ไม่เกี่ยวกับชุด Currency ใหม่
- **แยก News Layer จาก Financial State:** ความล้มเหลวชั่วคราวของ Calendar ไม่ควรหยุด Timer หรือการคำนวณ Drawdown/Profit Target ส่วนอื่น

### 4.2 ข้อจำกัดและ Failure Modes

1. **`NEWS_NO_EXPOSURE` ไม่เท่ากับ Safe to Trade**  
   หากบัญชีถือเฉพาะ Instrument ที่ Mapping ไม่รองรับ ระบบอาจไม่พบ Exposure ที่รู้จัก นี่ไม่ได้แปลว่าไม่มี Event Risk หรืออนุญาตให้ซื้อขายได้ ควรกำหนด Policy แยกสำหรับ Unsupported Instrument และคำสั่งใหม่

2. **Currency Membership ไม่ใช่ Quantitative Exposure**  
   การดู Base/Profit Currency เป็นเพียง Proxy แบบหยาบ ไม่ได้คำนึงถึงทิศทาง Position, Notional, Delta, Contract Multiplier, Conversion Rate, Option Greeks, Netting หรือความสัมพันธ์ระหว่างสินทรัพย์

3. **กรณี XAU/USD แคบเกินไป**  
   Gold Symbol ของ Broker อื่นอาจใช้ชื่อหรือ Metadata ต่างกัน นอกจากนี้ยังต้องพิจารณา Metals, Energy, Indices, Crypto CFDs, Futures และ Options

4. **Monitoring ไม่ใช่ Enforcement**  
   ส่วนนี้ไม่ได้ Block Order หรือ Close Position ต้องสร้าง Order-entry Gate แยกต่างหาก พร้อมกำหนด Fail-open/Fail-closed, Manual Override และ Race-condition Handling

5. **Calendar Metadata อาจไม่ครบ**  
   หาก `CalendarEventById()` ล้มเหลว ระบบอาจต้องถือว่า Search ของ Currency นั้น Unavailable แม้จะพบ Candidate อื่นแล้ว แนวทางนี้ค่อนข้าง Conservative แต่ต้องมี Policy รองรับ Downtime

6. **การเก็บ Event ที่เลือกเพียงรายการเดียวมีข้อจำกัด**  
   การเลือก Event ที่ Active ก่อน Event ในอนาคตเหมาะกับการรายงานสถานะปัจจุบัน แต่ UI/Audit Log ควรเก็บ Candidate ที่เกี่ยวข้องทั้งหมดหากต้องการตรวจสอบย้อนหลัง

7. **Cache 60 วินาทีมี Trade-off**  
   ลด API Calls แต่ Calendar Revision หรือ Cancellation อาจถูกตรวจพบช้า แม้เวลา Event ปกติจะทราบล่วงหน้า ควรทดสอบผลของ Cache Age ต่อการปฏิบัติงานจริง

8. **ยังไม่มี Operational Benchmark**  
   บทความไม่ได้รายงาน Calendar Latency, Timeout Rate, Memory, CPU หรือพฤติกรรมเมื่อมี Position และ Currency จำนวนมาก

9. **Manual Test ไม่ใช่หลักฐานความถูกต้องครบถ้วน**  
   ควรเพิ่ม Boundary Test, Fault Injection, Property-based Test และ Integration Test กับ Broker จริง

10. **Prop Firm Policy เปลี่ยนแปลงได้**  
    ค่า Default เช่น High Importance และ 10 นาทีก่อน/หลัง เป็นเพียงค่าตัวอย่าง ไม่ใช่ข้อยืนยันว่าตรงกับ Rulebook ปัจจุบันของทุกบริษัท ควร Version Policy แยกตาม Firm/Account

## 5. Backtest และหลักฐานเชิงประจักษ์

บทความนี้เป็นงาน Infrastructure ไม่ใช่ Signal Strategy และไม่ได้รายงาน:
- Equity Curve, Return Series หรือ Benchmark
- Win Rate, Expectancy, Sharpe, Sortino, Profit Factor หรือ Drawdown Comparison
- การลดความเสียหายจากข่าว
- False-positive / False-negative Rate
- Calendar API Latency หรือ Availability Statistics

ดังนั้น **ไม่สามารถสรุปว่าระบบสร้าง Trading Edge หรือเพิ่มผลตอบแทนได้** หลักฐานที่มีรองรับเพียงว่าผู้เขียนอธิบาย Implementation และแนวทางตรวจสอบ State ด้วยตนเอง ไม่ได้พิสูจน์ความถูกต้องกับทุก Broker, Symbol หรือ Live Failure Condition

## 6. แผน Validation ก่อนใช้งานจริง

### A. Symbol Coverage

สร้าง Broker-specific Test Matrix ครอบคลุม Forex Major/Minor, Symbol ที่มี Prefix/Suffix, Metals, Indices, Energies, Crypto CFDs, Futures และ Options โดยบันทึก Calculation Mode, Base/Profit Currency, Contract Size, Tick Value และ Expected Exposure Mapping

Instrument ที่ไม่รู้จักต้องมีสถานะ Unsupported/Unavailable อย่างชัดเจน ไม่ควรถูกตีความเป็น Safe โดยปริยาย

### B. State-machine Boundary Tests

ทดสอบเวลา:
- ตรงกับ (t_e - 60B)
- ก่อนและหลังจุดเริ่มต้น 1 วินาที
- ตรงกับ (t_e + 60A)
- หลังจุดสิ้นสุด 1 วินาที
- มี Event Active และ Event Upcoming ซ้อนกัน
- Event หลายสกุลเงินเกิดเวลาเดียวกัน
- Event ถูกยกเลิกหรือแก้ไขเวลา
- Trade Server Time เดินย้อนกลับ

### C. Fault Injection

จำลองความล้มเหลวของ Position Selection, Symbol Metadata, `CalendarValueHistory()`, `CalendarEventById()`, Timer Initialization และ Cache Invalidation แล้วตรวจสอบว่า Valid Empty ไม่กลายเป็น Error และ Error ไม่กลายเป็น `NEWS_NO_EVENT`

### D. Operational Tests

วัดจำนวน Calendar Calls, p50/p95/p99 Latency, Timeout Frequency, CPU/Memory และ Recovery Time หลังเกิด Transient Failure รวมถึงทดสอบกรณีมี Position และ Currency จำนวนมาก

### E. Enforcement Integration

กำหนด Action ที่ชัดเจนสำหรับแต่ละ State โดยแนวทาง Conservative เบื้องต้น:

- `NEWS_RESTRICTED`: ปฏิเสธ Order ใหม่ที่เพิ่มความเสี่ยง แต่ไม่ควรปิด Position เดิมอัตโนมัติ เว้นแต่ Rulebook ระบุไว้
- `NEWS_UNAVAILABLE`: Fail-closed สำหรับ Order ใหม่ที่เพิ่มความเสี่ยง พร้อมแจ้งเตือนและ Retry แบบมี Backoff
- `NEWS_NO_EXPOSURE`: ห้ามตีความว่า Safe ทั่วไป ต้องตรวจสอบ Instrument ของคำสั่งใหม่แยกต่างหาก
- `NEWS_SAFE` / `NEWS_NO_EVENT`: อนุญาตได้เมื่อผ่าน Risk Control และกฎอื่นทั้งหมดแล้วเท่านั้น

ควรตรวจสอบ State ซ้ำทันทีก่อนส่งคำสั่ง เพื่อลด Race Condition ระหว่าง Signal Generation กับ Execution

### F. Auditability

บันทึก Timestamp, Server-time Offset, Exposure Set, Unsupported Positions, Event ID/Name/Currency/Importance, Search Result, Cache Age, State Transition และ Policy Version เพื่อให้สามารถ Reconstruct เหตุการณ์ย้อนหลังได้

## 7. ตัวอย่างเชิงตัวเลข

สมมติว่าบัญชีถือ:
- EURUSD
- GBPUSD
- XAUUSD โดย Broker Metadata ระบุ Base เป็น XAU และ Profit เป็น USD

Currency Set ที่ได้คือ `{EUR, USD, GBP}`

สมมติ Event ที่ผ่านเงื่อนไข:
- EUR เวลา 14:00
- USD เวลา 14:20
- GBP เวลา 15:00

กำหนด Pre/Post Window อย่างละ 10 นาที:

- เวลา 14:05: Event EUR ยัง Active (สิ้นสุด 14:10) และมี Priority สูงกว่า Event USD ที่กำลังจะเข้า Pre-window
- เวลา 14:11: Event EUR หมดช่วงแล้ว แต่ Event USD อยู่ใน Pre-window 14:10–14:30 ดังนั้น Account State ยังคงเป็น `NEWS_RESTRICTED` เพียงแต่ Selected Event เปลี่ยนจาก EUR เป็น USD

ตัวอย่างนี้แสดงให้เห็นว่า Selected Event เปลี่ยนได้ ในขณะที่ Aggregate Restriction State ของบัญชียังคง Restricted ต่อเนื่อง

## 8. บทสรุปและข้อเสนอแนะเชิง Quant

บทความนำเสนอ Pattern ที่มีประโยชน์สำหรับ MQL5 ได้แก่ Account-wide Exposure Discovery, Explicit Result Enums, Calendar Event Ranking, Pre/Post Window Logic และ Cache Refresh

บทเรียนที่สำคัญที่สุดคือ **Valid Empty ไม่ใช่ Unavailable และ Unsupported Exposure ไม่ใช่ No Risk**

สำหรับ Quant Trading Infrastructure ควรใช้ระบบนี้เป็นจุดเริ่มต้นของ Policy-aware Monitoring Service ไม่ใช่ Risk Engine ที่สมบูรณ์ ควรขยาย Instrument Taxonomy, วัด Exposure ให้เหมาะกับสินทรัพย์, กำหนด Fail-open/Fail-closed, เชื่อม Enforcement ที่ Execution Boundary และทดสอบด้วย Deterministic Test กับ Operational Telemetry

ไม่มีหลักฐานเชิงประจักษ์ในบทความนี้ที่ยืนยันว่า Component ดังกล่าวสร้าง Alpha หรือเพิ่มผลตอบแทน

---

## แหล่งอ้างอิง

1. Chacha Ian Maroa, “Building a Prop Firm Compliance Monitor in MQL5 (Part 4): Economic Calendar and Account Exposure Monitoring”, 16 Nov 2026 — https://www.mql5.com/en/articles/24399
2. MQL5 Reference — Economic Calendar: https://www.mql5.com/en/docs/calendar
3. MQL5 Reference — CalendarValueHistory: https://www.mql5.com/en/docs/calendar/calendarvaluehistory
4. MQL5 Reference — TimeTradeServer: https://www.mql5.com/en/docs/dateandtime/timetradeserver
5. MQL5 Article — MetaTrader 5 and the MQL5 Economic Calendar: How to Turn News into a Reproducible Trading System: https://www.mql5.com/en/articles/22196

## Evidence Ledger

- **Author-described implementation:** State Model, Supported Symbol Mapping, Cache Interval และ Manual Test Matrix
- **ตรวจสอบกับ MQL5 Reference:** Economic Calendar ใช้ Trade-server Time และ `CalendarValueHistory()` คืนจำนวนข้อมูลหรือ `-1` เมื่อเกิด Error
- **Independent Analysis:** ข้อจำกัดของ Unsupported Symbols, Exposure Proxy, Fail-closed Policy, Monitoring-vs-Enforcement และการไม่มี Performance Evidence
- **ยังไม่ได้ตรวจสอบในงานวิจัยนี้:** การ Compile ไฟล์ `.mq5`, การรันกับ Broker จริง, ความถูกต้องกับ Symbol Specification ทุกประเภท และ Live Calendar Reliability
