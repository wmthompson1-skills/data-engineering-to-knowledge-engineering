In Infor Visual Manufacturing, mapping ISA-95 ontology objects directly to standard relational tables requires navigating a highly specific, transactional schema. Visual uses a relational, database-driven costing engine rather than an event-stream model.
For Actual FIFO Costing, Visual tracks layers natively by capturing historical operational costs and assigning exact dollar amounts to specific inventory receipt records.

1. ISA-95 to Infor Visual Table Mapping

To structure your ontology relationships, map the conceptual ISA-95 objects directly to the primary physical tables in Visual:

ISA-95 Concept	Infor Visual ERP Table	Description / Purpose
Material Class / Definition	PART	Master record for part numbers, SKUs, and standard inventory parameters.
Material Lot	LOT_TRACK	Contains specific physical lot numbers and batch metadata.
Raw Materials Inventory	PART_LOCATION	Material inventory records where PART.TYPE = 'P' (Purchased).
Work in Process (WIP)	WORK_ORDER, WO_OPERATION	Tracks jobs, operations, labor hours, and issuing overhead.
Finished Goods Inventory	PART_LOCATION	Material inventory records where PART.TYPE = 'M' (Manufactured).
General Ledger Triggers	TRANS_LOG	The master transactional audit trail. Everything hitting the GL passes here.


2. Implementing Actual FIFO Costing via the Schema

In an actual FIFO system, you cannot rely purely on the current stock balance; you must trace the active unrelieved inventory layers.
Visual implements this via tracking tables that handle the sequential clearing of layers. Your ontology queries should look at the following layers:
[INVENTORY_FIFO_LAYER]  <-- Tracks remaining unconsumed quantity and unit cost per layer
         |
[INVENTORY_TRANSACTION] <-- The link connecting the physical movement to the FIFO cost layer
         |
    [TRANS_LOG]         <-- Generates the financial entry strings for the GL
• INVENTORY_FIFO_LAYER (or similar core tracking views): This is where Visual maintains the queue of received lots, their initial quantities, remaining quantities, and actual purchase order/manufacturing actual costs.
• The Valuation Link: When a transaction occurs, Visual pulls cost from the oldest active record in this table for that specific PART_ID.

3. ISA-95 Material Events mapped to Visual GL Queries

To build semantic wrappers or ETL views over Visual for your ontology, query the TRANS_LOG table. Look for specific transaction types (TYPE) to reconstruct the ISA-95 material lifecycle and identify the corresponding GL impacts.

Event 1: Raw Material Issued to a Work Order (ISA-95 Material Consumed)

• Visual Logic: Material is issued from inventory to a specific Work Order operation.
• Table Fields to Query: TRANS_LOG where TYPE = 'I' (Issue)
• Ontology Mapping Data:
	• Part/Lot Identification: TRANS_LOG.PART_ID, TRANS_LOG.LOT_ID
	• Cost Vector: TRANS_LOG.ACTUAL_COST (Calculated by Visual using the FIFO layer pulled during execution)
	• Financial Flow: Debit WORK_ORDER.WIP_ACCT_ID | Credit PART.INV_ACCT_ID

Event 2: Value-Add Activities / In-Process Labor (ISA-95 Personnel/Equipment Performance)

• Visual Logic: Labor hours are charged to an operational piece of equipment or work center, adding value to WIP.
• Table Fields to Query: LABOR_TICKET or TRANS_LOG where TYPE = 'L' (Labor) or TYPE = 'O' (Overhead)
• Ontology Mapping Data:
	• WIP Identity: Linked via TRANS_LOG.WORKORDER_BASE_ID
	• Financial Flow: Debit Work Order WIP Account | Credit Labor Payroll Clearing / Overhead Absorption Account

Event 3: Finished Goods Production Receipt (ISA-95 Material Produced)

• Visual Logic: A work order completes, relieving WIP and creating a brand new manufactured FIFO layer in Finished Goods.
• Table Fields to Query: TRANS_LOG where TYPE = 'F' (Finished Good Receipt)
• Ontology Mapping Data:
	• Part/Lot Identification: TRANS_LOG.PART_ID (where PART.TYPE = 'M')
	• Cost Vector: TRANS_LOG.ACTUAL_COST (The cumulative actual material + labor + overhead rolled up from the job)
	• Financial Flow: Debit PART.INV_ACCT_ID (Finished Goods Asset) | Credit WORK_ORDER.WIP_ACCT_ID

4. Semantic Query Design Pattern

If you are mapping this to an ontology layer (like a semantic view or virtual graph), write your abstraction queries to resolve relationships dynamically.
For instance, to look up the exact Actual FIFO Cost of an active ISA-95 Finished Goods Lot, your semantic graph query logic must execute a relational join equivalent to this:
sql
SELECT 
    tl.PART_ID AS MaterialDefinitionID,
    tl.LOT_ID AS MaterialLotID,
    tl.TRANSACTION_DATE AS EventTimestamp,
    tl.QTY AS Quantity,
    tl.ACTUAL_COST AS ActualFIFOCost,
    p.INV_ACCT_ID AS GL_AssetAccount
FROM 
    TRANS_LOG tl
JOIN 
    PART p ON tl.PART_ID = p.ID
WHERE 
    tl.TYPE = 'F'               -- ISA-95: Material Produced
    AND p.TYPE = 'M'            -- Ontological Classification: Finished Good
    AND tl.TRANSACTION_DATE >= '2026-01-01';
Use code with caution.
To finalize the mapping layer for Visual Manufacturing, let me know:
• Are you writing SQL Views/Stored Procedures to feed this ontology, or using an external ETL tool?
• Do you track costs by Warehouse/Location (PART_LOCATION), or do you pool FIFO layers globally across the plant?
