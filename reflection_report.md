# Munder Difflin Multi-Agent System: Reflection Report

This document provides an overview of the design, evaluation, and future improvements for the Munder Difflin multi-agent system.

---

## 1. Workflow Diagram and Agent Architecture

The system is built around a classic **Orchestrator (or Router)** pattern, which manages a team of specialized agents. The workflow is as follows:

1.  **Customer Request**: A text-based request from a customer is the entry point.
2.  **Orchestrator Agent**: This agent analyzes the user's request to determine its primary intent (e.g., ordering, quoting, or checking inventory). It uses simple keyword matching for routing.
3.  **Specialist Agents**: The request is delegated to one of three specialist agents:
    *   **Inventory Agent**: Handles all queries related to stock levels, item availability, and restock needs. Its tools primarily perform `SELECT` queries on the database.
    *   **Quoting Agent**: Responsible for generating price quotes. It fetches item prices, calculates totals, and applies a hardcoded bulk discount rule (10% for quantities >= 100).
    *   **Ordering Agent**: Processes actual purchase requests. It validates stock availability and, if successful, creates a `sales` transaction in the database, which financially impacts the system.
    *   **Notification Agent**: A non-core agent that can be triggered to send communications like order confirmations or low-stock alerts. In the current implementation, it prints messages to the console.
4.  **Response Generation**: The specialist agent returns a structured result (or a string, in earlier versions) to the orchestrator.
5.  **Final Output**: The orchestrator formats this result into a customer-facing response and returns it.

The ASCII diagram included in `project_final.py` provides a visual representation of this flow.

### Why This Architecture?

I chose this orchestrator-based architecture for several key reasons:

*   **Modularity and Separation of Concerns**: Each agent has a single, well-defined responsibility. This makes the system easier to develop, debug, and maintain. For example, if the quoting logic needs to change, only the `Quoting Agent` and its tools need to be modified.
*   **Scalability**: It is easy to add new capabilities by simply creating a new specialist agent and updating the orchestrator's routing logic. For instance, a "Business Intelligence Agent" could be added without affecting the existing agents.
*   **Efficiency**: By routing requests to the correct specialist, we avoid unnecessary tool calls or complex, monolithic prompts. The agent assigned to a task has exactly the tools it needs to succeed.

---

## 2. Analysis of Evaluation Results

The `test_results.csv` file reveals critical insights into the system's performance:

*   **High Failure Rate**: The vast majority of requests failed. The primary reasons cited in the responses are "item not found" or "insufficient stock."
*   **Single Point of Success**: Only one request (ID #3) was partially fulfilled, resulting in a change to the `cash_balance`. This confirms the `Ordering Agent` and transaction system work correctly when an item is available.
*   **Inventory Limitations**: The core issue stems from the initial database setup in `init_database`, which only stocks a random 40% of the items from the master `paper_supplies` list. Many customer requests are for items that simply do not exist in the `inventory` table.
*   **Brittle Item Matching**: The system requires an exact match between the customer's request (e.g., "A4 white printer paper") and the item name in the database (e.g., "A4 paper"). This brittleness is a major contributor to the "item not found" errors.

In summary, the agent orchestration and transaction logic are fundamentally sound, but the system's practical effectiveness is severely hampered by data limitations and a lack of robust entity matching.

---

## 3. Future Improvements

Based on the evaluation, here are two high-impact improvements for the future:

1.  **Implement Fuzzy Item Matching**: To overcome the "item not found" issue, the system needs to intelligently map customer descriptions to canonical item names. This could be achieved by:
    *   Creating a dedicated `Item Recognition Agent` that uses semantic search or embedding-based similarity to find the closest match in the `inventory` table before passing the canonical name to other agents.
    *   Fine-tuning a smaller model to act as a lightweight entity extractor and normalizer.

2.  **Develop a Proactive Restocking Agent**: The current system is purely reactive. A proactive agent could significantly improve stock availability. This agent would:
    *   Run on a schedule (e.g., at the end of each simulated day).
    *   Use the `inventory_check_restock_need` tool to identify all items below their `min_stock_level`.
    *   Automatically create `stock_orders` transactions to replenish inventory, simulating a purchase from a supplier. This would make the system more resilient and capable of fulfilling more orders over time.

An additional improvement would be to enhance the **Business Advisor Agent** concept. This agent could analyze long-term sales trends from the `transactions` table to recommend adjustments to `unit_price` or `min_stock_level` for items, optimizing for profitability and availability.

---