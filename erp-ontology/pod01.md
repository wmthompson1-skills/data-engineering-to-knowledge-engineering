To build an ontology for manufacturing inventory, Work in Process (WIP), Finished Goods (FG), and General Ledger (GL) integrations using ISA-95 (IEC 62264), you must map physical material flows to financial accounting structures. [1] (https://rhize.com/blog/what-is-isa95/)
ISA-95 models physical and operational entities, while the General Ledger handles financial entities. The bridge between them is ISA-95 Part 2 (Material Model) and Part 4 (Work Master / Operations Performance). [1] (https://www.linkedin.com/pulse/isa-95iec-62264-standard-manufacturing-operations-management-abc8c), [2] (https://www.youtube.com/watch?v=NTzHpmM4clU)

1. Architectural Alignment (The Level 3 to Level 4 Boundary)

Under the ISA-95 Purdue Reference Architecture, financials sit at Level 4 (ERP), while material transformations sit at Level 3 (MES/MOM). [1] (https://connect981.com/faqs/what-is-the-isa-95-purdue-model), [2] (https://rhize.com/blog/what-is-isa95/)
+---------------------------------------------------------------------------------+

| Level 4: Enterprise (ERP)                                                       |
|   -> General Ledger Accounts (Asset, COGS, Inventory Balances)                   |
+---------------------------------------------------------------------------------+
                                       ||  B2MML / API Boundary
+---------------------------------------------------------------------------------+

| Level 3: Manufacturing Operations (MES)                                         |
|   -> Material Lots, Sub-lots, Equipment Segments, Work Performance             |
+---------------------------------------------------------------------------------+

2. ISA-95 Core Material Object Mapping

To represent inventory states, leverage the ISA-95 Material Model classes: [1] (https://rhize.com/blog/what-is-isa95/)
• Material Class: The generic definition of the item (e.g., "Steel Coil", "Resin", "Assembled Widget").
• Material Definition: The specific part number or SKU (e.g., SKU-9982).
• Material Lot: A uniquely identifiable, trackable quantity (e.g., Batch-A101).
• Material Sub-Lot: A broken-down portion of a lot used for specific tracking (e.g., Pallet-3 of Batch-A101).

Mapping Accounting States to ISA-95 Material Properties

ISA-95 tracks states natively using the Material Lot Property attribute.

Inventory Financial State	ISA-95 Ontological Entity	Material Lot Property / State Values
Raw Materials (RM)	MaterialLot	Type: Raw, Status: Available / Inspected
Work in Process (WIP)	MaterialLot + ProductionStep	Type: WIP, Status: In-Progress, Linked to a active EquipmentElement or WorkCell
Finished Goods (FG)	MaterialLot	Type: Finished, Status: Released / Staged


3. Ontological Relationship & Ledger Triggers

To pass clean data to the General Ledger, Level 3 operations must emit ISA-95 Material Performance and Production Performance events. These events dictate double-entry bookkeeping movements. [1] (https://www.linkedin.com/pulse/isa-95iec-62264-standard-manufacturing-operations-management-abc8c)
[Raw Material Lot] ===(Material Consumption Event)===> [WIP Lot] ===(Production Completion Event)===> [Finished Goods Lot]

       |                                                    |                                                |
  GL Debit: WIP                                        GL Debit: Finished Goods                        GL Debit: COGS
  GL Credit: Raw Materials                             GL Credit: WIP                                  GL Credit: Finished Goods

Event 1: Raw Material Issue to Production

• Operational Trigger: A material handler scans a MaterialLot into a Level 3 WorkCell. [1] (https://explitia.com/blog/isa-95-explained-mes-erp-manufacturing-data/)
• ISA-95 Data Element: MaterialPerformance -> MaterialConsumed. [1] (https://www.linkedin.com/pulse/isa-95iec-62264-standard-manufacturing-operations-management-abc8c)
• GL Mapping Object:
	• Debit: WIP Inventory Account (Asset)
	• Credit: Raw Materials Inventory Account (Asset)

Event 2: Material Transformation (WIP / Value Add)

• Operational Trigger: Labor hours or machine run-times are compiled.
• ISA-95 Data Element: PersonnelPerformance and EquipmentPerformance. [1] (https://www.linkedin.com/pulse/isa-95iec-62264-standard-manufacturing-operations-management-abc8c)
• GL Mapping Object:
	• Debit: WIP Inventory Account (Direct Labor / Overhead standard costs capitalized)
	• Credit: Payroll Clearing / Factory Overhead Absorption Accounts

Event 3: Finished Goods Production Receipt

• Operational Trigger: Final quality sign-off and receipt of final item to the warehouse. [1] (https://explitia.com/blog/isa-95-explained-mes-erp-manufacturing-data/)
• ISA-95 Data Element: MaterialPerformance -> MaterialProduced. [1] (https://www.linkedin.com/pulse/isa-95iec-62264-standard-manufacturing-operations-management-abc8c)
• GL Mapping Object:
	• Debit: Finished Goods Inventory Account (Asset)
	• Credit: WIP Inventory Account (Relieving WIP)

4. Technical Implementation Design Patterns

When building this into a graph-based ontology (like OWL/RDF) or a Unified Namespace (UNS) over MQTT, use this standard schema structure: [1] (https://www.youtube.com/watch?v=NTzHpmM4clU), [2] (https://www.youtube.com/watch?v=WsZTEe6nXMc&t=221)
json
{
  "Enterprise": "GlobalCorp",
  "Site": "AuburnPlant",
  "Area": "Assembly",
  "ISA95_Model": "MaterialPerformance",
  "Event_Timestamp": "2026-10-29T22:13:00Z",
  "Payload": {
    "MaterialDefinitionID": "SKU-9982",
    "MaterialLotID": "LOT-2026-1029",
    "Quantity": 150,
    "UnitOfMeasure": "kg",
    "MaterialUse": "Consumed", 
    "FinancialContext": {
      "GL_SourceAccount_Hint": "12100-RawMaterials",
      "GL_TargetAccount_Hint": "12200-WIP"
    }
  }
}
Use code with caution.
Tip: Avoid hardcoding specific General Ledger account strings inside your Level 3 MES ontology. Instead, ensure your Level 3 ontology exposes semantic tags—such as MaterialUse: Consumed, MaterialClass: Metal, or CostCenter: Line1—allowing the ERP's accounting rules engine to dynamically determine the exact GL account string.
To refine the semantic rules for your ontology, let me know:
• What database architecture are you using? (Graph DB, Unified Namespace/MQTT, or standard relational ERP tables?)
• Do you use standard costs, or do you require actual/FIFO costing tracking down to the individual lot?
