# Professional Hotel Management Tool
A high-performance desktop application built with **C++** and the **Qt Framework**, designed to demonstrate efficient data management through custom-implemented data structures.

## 🛠 Technical Highlights
* **Binary Search Tree (BST):** Engineered a custom BST for room record management, enabling $O(\log n)$ search, insertion, and retrieval based on room numbers.
* **Singly Linked List:** Implemented a dynamic linked list to manage customer profiles, ensuring flexible memory allocation.
* **Merge Sort Algorithm:** Developed a recursive Merge Sort for the customer linked list to achieve optimal $O(n \log n)$ alphabetical sorting of guest records.
* **Event-Driven Architecture:** Leveraged **Qt’s Signals & Slots** mechanism to create a responsive and decoupled User Interface.

## 📂 Project Structure
* `/src`: Core C++ source files (`.cpp`, `.h`).
* `main.moc`: Meta-Object Compiler file for Qt event handling.
* `Technical_Report.pdf`: Comprehensive analysis of algorithmic complexity and system design.

## 🚀 Key Features
* **Real-time Search:** Integrated search functionality for rooms and customer IDs.
* **Dynamic Booking:** automated room availability updates and check-in/check-out workflows.
* **Data Visualization:** Grouped booking views utilizing `QMap` for sorted data representation.
