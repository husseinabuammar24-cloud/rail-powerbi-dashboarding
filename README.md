# RailPulse Analytics: Transit Performance Dashboard

- **Repository:** `challenge-powerbi`
- **Type of Challenge:** `Learning`
- **Team Challenge:** `Solo`
- **Deployment Strategy:** Power BI Service (Azure SQL Integration)

---

## 🔗 Project Links & Deliverables

* **Power BI Service Dashboard Link:** [RailPulse Performance Dashboard](https://app.powerbi.com/groups/0574653e-aeab-44e3-8b67-6e20c0a1ded5/reports/d09652e9-47c6-4ff5-93d9-992d6ad67b31/20507b7183086048142e?experience=power-bi)
* **Workbook File:** `RailPulse Dataset Report.pbix` (committed in root directory)

---

## 📊 Visual Design Rationale & Core Insights

The dashboard is structured around an executive-first layout to provide immediate visibility into train performance and operational bottlenecks:

1. **Punctuality & Operational KPIs (Top Banner):** 
   - Displays high-level executive cards for **On-Time Rate %**, **Cancellation Rate %**, **Average Delay**, **Total Delayed Minutes**, and **Total Trains**.
2. **Train Class Performance (Bottom Left):** 
   - Breaks down **Total Delayed Minutes by Train Class** to isolate high-impact train types. InterCity (`IC`) trains drive the majority of network delay minutes[cite: 2].
3. **Platform Congestion Map (Bottom Middle):** 
   - Categorizes total volume across platform assignments[cite: 1]. Platforms 4, 3, and 5 handle disproportionately high volume compared to higher platform numbers[cite: 1].
4. **Rush Hour Matrix (Bottom Right):** 
   - Evaluates **Train Count vs. Avg Delay (min) by Hour**[cite: 1]. Highlights clear network volume spikes during morning rush hours (6 AM – 8 AM)[cite: 1].

---

## 🧮 Data Model & DAX Formulas

The data model uses a clean Star Schema connecting `liveboard_records` (Fact) to `stations` and `vehicles` (Dimensions)[cite: 2]. 

### Key DAX Measures

```dax
On-Time Rate % = 
DIVIDE(
    CALCULATE(COUNTROWS(liveboard_records), liveboard_records[delay_seconds] < 120, liveboard_records[canceled] = FALSE),
    CALCULATE(COUNTROWS(liveboard_records), liveboard_records[canceled] = FALSE)
)

Cancellation Rate % = 
DIVIDE(
    CALCULATE(COUNTROWS(liveboard_records), liveboard_records[canceled] = TRUE()),
    COUNTROWS(liveboard_records),
    0
)

Avg Delay (min) = AVERAGE(liveboard_records[delay_seconds]) / 60

Total Delayed Minutes = SUM(liveboard_records[delay_seconds]) / 60


📝 Methodology Note — On-Time Rate %
Canceled trains are excluded from both the numerator and denominator of On-Time Rate %. This measures punctuality among trains that actually ran, keeping it comparable to standard rail industry KPIs. Cancellations are tracked separately as a distinct metric.

💡 Top 3 Tactical Recommendations for SNCB/NMBS
Spread Out Platform Usage: Platforms 4, 3, and 5 take the heaviest load—move some trains to less busy tracks during rush hour to stop station bottlenecks[cite: 1].

Add Extra Buffer Time for IC Trains: Give InterCity (IC) trains a 2-minute cushion in their schedules so small delays don't cause a chain reaction across other lines[cite: 1].

Focus Crew Support on Morning Rush Hours: Put extra dispatch and support staff on duty between 6 AM and 8 AM, when train volume peaks, to resolve delays instantly[cite: 1].
