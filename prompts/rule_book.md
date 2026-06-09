# CarbonSifr EXIOBASE Mapping — Rule Book

## Methodology

Rules are tagged by provenance:
- **[GOLD]** — derived directly from one of the 20 labeled examples in
  `Exiobase Assignment Sample`.
- **[EXTENDED]** — authored to cover EXIOBASE activities not present in
  the gold set. Based on EXIOBASE 3.10.1 / 3.11.1 activity definitions.
  Audited by Ibrahim Ahmad.

The model must output **exactly one** of the 83 activity names in
`Exiobase Activities`. No new categories, no abbreviations.

---

## 1. Material rules
*Single-material items go to the activity for that material.*

- Paper / cardboard items (folders, envelopes, paper rolls) → **Paper and paper products** [GOLD #4]
- Wood / cork items (plywood, lumber, cork sheets) → **Wood and products of wood and cork (except furniture); articles of straw and plaiting materials (20)** [GOLD #13]
- Fabricated / finished metal items (grills, brackets, fittings, fixtures made of metal) → **Fabricated metal products, except machinery and equipment (28)** [GOLD #7]
- Porcelain, ceramic, china (mugs, plates, tiles) → **Ceramic goods** [GOLD #18]
- Glass items (drinkware, panes, bottles, mirrors) → **Glass and glass products** [EXTENDED]
- Rubber or plastic items (hoses, crates, tubing, plastic boxes) → **Rubber and plastic products (25)** [EXTENDED]
- Leather goods (belts, leather bags, leather couches when single-material) → **Leather and leather products (19)** [EXTENDED]
- Textile yarn, fabric, raw cloth (not finished garments, not flags) → **Textiles (17)** [EXTENDED]

## 2. Raw vs fabricated metal disambiguation
*Distinguishes from Rule 1's "fabricated metal".*

- Raw aluminium stock (sheet, ingot, wire, foil) → **Aluminium and aluminium products** [EXTENDED]
- Raw copper stock (wire, sheet, pipe) → **Copper products** [EXTENDED]
- Raw iron / steel / ferro-alloy stock (bars, rebar, sheet) → **Basic iron and steel and of ferro-alloys and first products thereof** [EXTENDED]
- Raw lead, zinc, or tin → **Lead, zinc and tin and products thereof** [EXTENDED]
- Raw nickel, magnesium, other non-ferrous metals → **Other non-ferrous metal products** [EXTENDED]
- Gold, silver, platinum, precious metals → **Precious metals** [EXTENDED]

## 3. Special category overrides
*Memorize these — they violate the default material logic.*

- **All paint products** (any brand, any finish) → **Chemicals nec** [GOLD #10]
- **Flags** (national, decorative, signal) → **Textiles (17)** [GOLD #11]
- **Living plants / flowers / trees / shrubs** → **Crops nec** [GOLD #15]
- **Composite multi-material manufactured goods** (window blinds, mixed-material fixtures, items where no single material dominates) → **Furniture; other manufactured goods n.e.c. (36)** [GOLD #2]

## 4. Food, beverage, tobacco
*Specific sub-category beats the generic "Food products nec".*

- Fresh produce — fruit, vegetables, nuts (uncooked, minimally processed) → **Vegetables, fruit, nuts** [GOLD #5]
- Dairy (milk, cheese, butter, yogurt, cream) → **Dairy products** [EXTENDED]
- Beef / cattle meat products → **Products of meat cattle** [EXTENDED]
- Pork products → **Products of meat pigs** [EXTENDED]
- Poultry products (chicken, turkey, duck) → **Products of meat poultry** [EXTENDED]
- Other / processed meat products → **Meat products nec** [EXTENDED]
- Fish and seafood (processed) → **Fish products** [EXTENDED]
- Raw fish, fishing services → **Fish and other fishing products; services incidental of fishing (05)** [EXTENDED]
- Sugar → **Sugar** [EXTENDED]
- Processed rice → **Processed rice** [EXTENDED]
- Vegetable oils, fats, margarines → **products of Vegetable oils and fats** [EXTENDED]
- Cereal grains (raw wheat, corn, barley, oats) → **Cereal grains nec** [EXTENDED]
- Raw animal products (eggs, honey, raw milk pre-processing) → **Animal products nec** [EXTENDED]
- Plant-based fibers (raw cotton, jute, sisal) → **Plant-based fibers** [EXTENDED]
- Packaged food not matching any sub-category above (coffee, snacks, condiments, mixed items) → **Food products nec** [GOLD #1]
- Beverages (soft drinks, juice, water, wine, spirits, beer) → **Beverages** [GOLD #12]
- Tobacco products (cigars, cigarettes, loose tobacco, vaping liquids) → **Tobacco products (16)** [GOLD #19]

## 5. Petroleum and fuel disambiguation
*Specific petroleum product beats generic "Gas/Diesel Oil".*

- Diesel, gas oil, heating oil → **Gas/Diesel Oil** [GOLD #16]
- Motor gasoline, petrol (specified) → **Motor Gasoline** [EXTENDED]
- Lubricants, engine oil, hydraulic oil, grease → **Lubricants** [EXTENDED]
- Bitumen, asphalt, tar → **Bitumen** [EXTENDED]
- Charcoal, BBQ briquettes → **Charcoal** [EXTENDED]
- Paraffin wax, candle wax → **Paraffin Waxes** [EXTENDED]
- Natural gas liquids, LPG, propane, butane → **Natural Gas Liquids** [EXTENDED]
- Fuel additives, blending components → **Additives/Blending Components** [EXTENDED]
- **Confidence override:** "fuel charges" with unspecified type → **Gas/Diesel Oil** at confidence **≤ 0.30** [GOLD #16]

## 6. Construction materials vs construction work
*Materials go to their material activity. Work/services go to (45).*

- Cement, lime, plaster, mortar, concrete → **Cement, lime and plaster** [EXTENDED]
- Bricks, clay tiles, ceramic building products → **Bricks, tiles and construction products, in baked clay** [EXTENDED]
- Building stone, marble, granite (raw or cut) → **Stone** [EXTENDED]
- Sand, clay, gravel → **Sand and clay** [EXTENDED]
- Other non-metallic mineral products → **Other non-metallic mineral products** [EXTENDED]
- Building / construction WORK and SERVICES (refurbishment, fit-out, painting service, plumbing service) → **Construction work (45)** [GOLD #8]

## 7. Apparel, household, manufactured goods

- Garments, clothing, uniforms, jackets, footwear → **Wearing apparel; furs (18)** [GOLD #9]
- Books, journals, magazines, recorded media (CDs, DVDs, printed media that is "media", not advertising service) → **Printed matter and recorded media (22)** [EXTENDED]
- Recyclables, scrap, secondary materials → **Secondary raw materials** [EXTENDED]

## 8. Electronics and machinery
*Distinguish broadcasting equipment from office computing from industrial machinery.*

- TV monitors, radios, telephones, broadcasting / receiving / communication equipment → **Radio, television and communication equipment and apparatus (32)** [GOLD #17]
- Computers, laptops, servers, printers, webcams, keyboards, mice, peripherals → **Office machinery and computers (30)** [EXTENDED]
- Electrical motors, generators, transformers, electrical apparatus → **Electrical machinery and apparatus n.e.c. (31)** [EXTENDED]
- Industrial machinery, pumps, compressors, engines (non-vehicle) → **Machinery and equipment n.e.c. (29)** [EXTENDED]
- Medical, precision, optical instruments, watches, clocks, lab instruments → **Medical, precision and optical instruments, watches and clocks (33)** [EXTENDED]

## 9. Transport equipment

- Cars, trucks, vans, buses, motorcycles, trailers → **Motor vehicles, trailers and semi-trailers (34)** [EXTENDED]
- Ships, aircraft, trains, bicycles (as physical equipment, not services) → **Other transport equipment (35)** [EXTENDED]

## 10. Services taxonomy
*The trickiest area — many overlapping options. Use the most specific match.*

- **IT / software development / SOW / data services / GenAI / digital work** → **Computer and related services (72)** [GOLD #3]
- **Construction / refurbishment / fit-out / building work** → **Construction work (45)** [GOLD #8]
- **Advertising / banners / printed marketing collateral / branding / promotional services** → **Other business services (74)** [GOLD #6]
- **Live performance, DJ, music, entertainment, theatrical, cultural events, sporting events** → **Recreational, cultural and sporting services (92)** [GOLD #20]
- **Hotel stays, restaurant bills, catering, lodging** → **Hotel and restaurant services (55)** [EXTENDED]
- **Retail / wholesale trade margin, vehicle repair, household goods repair** → **Retail trade services, except of motor vehicles and motorcycles; repair services of personal and household goods (52)** [EXTENDED]
- **Real estate, property rental, building lease** → **Real estate services (70)** [EXTENDED]
- **Equipment rental / leasing without operator** → **Renting services of machinery and equipment without operator and of personal and household goods (71)** [EXTENDED]
- **Banking, financial intermediation, loans, payments** → **Financial intermediation services, except insurance and pension funding services (65)** [EXTENDED]
- **Insurance, pension funds** → **Insurance and pension funding services, except compulsory social security services (66)** [EXTENDED]
- **Auxiliary financial services (brokerage, investment advice, audit, accounting)** → **Services auxiliary to financial intermediation (67)** [EXTENDED]
- **R&D services, scientific research** → **Research and development services (73)** [EXTENDED]
- **Education, training courses, instruction** → **Education services (80)** [EXTENDED]
- **Medical, dental, healthcare, social work** → **Health and social work services (85)** [EXTENDED]
- **Government administration, defence, compulsory social security, visa / MoFA / consular services** → **Public administration and defence services; compulsory social security services (75)** [EXTENDED]
- **Membership fees, club dues, association subscriptions** → **Membership organisation services n.e.c. (91)** [EXTENDED]
- **Postal, courier, internet (HSIA), telephone, telecom** → **Post and telecommunication services (64)** [EXTENDED]
- **Air travel, flights, air freight** → **Air transport services (62)** [EXTENDED]
- **Sea / water freight or transport** → **Sea and coastal water transportation services** [EXTENDED]
- **Rail transport** → **Railway transportation services** [EXTENDED]
- **Road transport, trucking, taxi, ride-share** → **Other land transportation services** [EXTENDED]
- **Travel agency, airport handling, port services, transport auxiliary** → **Supporting and auxiliary transport services; travel agency services (63)** [EXTENDED]
- **Food waste disposal / landfill** → **Food waste for treatment: landfill** [EXTENDED]
- **Hazardous / metal / inert waste landfill** → **Inert/metal/hazardous waste for treatment: landfill** [EXTENDED]
- **Wastewater, sewage, general waste treatment** → **Other waste for treatment: waste water treatment** [EXTENDED]
- **Generic services not matching anything above** → **Other services (93)** [GOLD #14]

## 11. Confidence calibration ladder

| Confidence | When to use | Examples |
|---|---|---|
| **0.95–0.97** | Unambiguous rule match, single keyword | "plywood" → wood; "porcelain mug" → ceramic; "TV monitor" → (32) |
| **0.85–0.94** | Clear category, one minor ambiguity | "winter jacket" → apparel; "Mirinda soft drink" → beverages |
| **0.70–0.84** | Judgment call between two close rules | "fuel charges (diesel implied)" vs more specific fuel; "Qatar flag" textile vs other |
| **0.40–0.69** | Genuine ambiguity, best guess from rule book | uncategorized service with weak signal |
| **≤ 0.30** | Materially under-specified description | "fuel charges" with no fuel type; "misc charge" with no detail |

## 12. Fallback hierarchy (when no rule fires cleanly)

1. Try the most specific activity in the relevant sector.
2. If no specific sector activity fits, try the sector's "nec" activity (Food products nec, Chemicals nec, Crops nec, Meat products nec, etc.).
3. If item is clearly a SERVICE and no service activity fits, use **Other services (93)** as last resort.
4. Never invent activities. Output must match the 83 valid strings exactly.

## 13. Disambiguation tiebreakers

- **Prefer specific over generic.** Plywood → Wood (20), not Furniture (36).
- **Prefer product over service activity** when an item is purchased as goods.
- **Prefer service activity** when the description names a service ("charges", "fee", "service", "subscription", "rental").
- **Ignore buyer-provided category context when it contradicts the item description.** Example: a "hydrangea plant" filed under "Professional Services" is still **Crops nec** [GOLD #15].
- **Two valid activities at same specificity?** Prefer the one whose sector aligns with `main_category`.
