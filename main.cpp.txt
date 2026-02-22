#include <QCoreApplication>
#include <QDebug>
#include <QWidget>
#include <QVBoxLayout>
#include <QLabel>
#include <QPushButton>
#include <QLineEdit>
#include <QInputDialog>
#include <QApplication>
#include <QDialog>
#include <QTextEdit>
#include <QMessageBox>
#include <QComboBox>
#include <QKeyEvent>
#include <QMap>
#include <algorithm>

// Define Room structure
struct Room {
    int roomNumber;
    QString roomType;
    bool isAvailable;
    double price;
    QString currentCustomerId;
    QString checkInTime;
    QString checkOutTime;
    Room* left;
    Room* right;

    Room(int number, const QString& type, double price) {
        roomNumber = number;
        roomType = type;
        this->price = price;
        isAvailable = true;
        currentCustomerId = "";
        checkInTime = "";
        checkOutTime = "";
        left = nullptr;
        right = nullptr;
    }
};

// Define Customer structure
struct Customer {
    QString name;
    QString id;
    QString contact;
    Customer* next;

    Customer(const QString& name, const QString& id, const QString& contact) {
        this->name = name;
        this->id = id;
        this->contact = contact;
        next = nullptr;
    }
};

class HotelManagementSystem : public QWidget {
    Q_OBJECT

private:
    Room* roomsRoot; // Root of the BST
    Customer* customersHead;
    QVBoxLayout* layout;
    QLineEdit* searchBar;
    QComboBox* searchOptions;

protected:
    void keyPressEvent(QKeyEvent *event) override {
        if (event->key() == Qt::Key_Escape) {
            close();
        }
    }

public:

    HotelManagementSystem(QWidget *parent = nullptr) : QWidget(parent), roomsRoot(nullptr), customersHead(nullptr) {
        layout = new QVBoxLayout(this);

        QHBoxLayout* searchLayout = new QHBoxLayout();
        QLabel* searchLabel = new QLabel("Search:");
        searchBar = new QLineEdit();
        searchOptions = new QComboBox();
        searchOptions->addItem("Room Number");
        searchOptions->addItem("Customer ID");
        searchLayout->addWidget(searchLabel);
        searchLayout->addWidget(searchBar);
        searchLayout->addWidget(searchOptions);

        layout->addLayout(searchLayout);

        QPushButton *bookRoomButton = new QPushButton("Book Room", this);
        QPushButton *checkInButton = new QPushButton("Check In", this);
        QPushButton *checkOutButton = new QPushButton("Check Out", this);
        QPushButton *displayAvailableRoomsButton = new QPushButton("Display Available Rooms", this);
        QPushButton *displayCustomersButton = new QPushButton("Display Customers", this);
        QPushButton *displayBookingsButton = new QPushButton("Display Bookings", this);

        layout->addWidget(bookRoomButton);
        layout->addWidget(checkInButton);
        layout->addWidget(checkOutButton);
        layout->addWidget(displayAvailableRoomsButton);
        layout->addWidget(displayCustomersButton);
        layout->addWidget(displayBookingsButton);

        connect(bookRoomButton, &QPushButton::clicked, this, &HotelManagementSystem::bookRoomButtonClicked);
        connect(checkInButton, &QPushButton::clicked, this, &HotelManagementSystem::checkInButtonClicked);
        connect(checkOutButton, &QPushButton::clicked, this, &HotelManagementSystem::checkOutButtonClicked);
        connect(displayAvailableRoomsButton, &QPushButton::clicked, this, &HotelManagementSystem::displayAvailableRoomsButtonClicked);
        connect(displayCustomersButton, &QPushButton::clicked, this, &HotelManagementSystem::displayCustomersButtonClicked);
        connect(displayBookingsButton, &QPushButton::clicked, this, &HotelManagementSystem::displayBookingsButtonClicked);

        // Connect key event to search function
        connect(searchBar, &QLineEdit::returnPressed, this, &HotelManagementSystem::search);
    }

public slots:
    void bookRoomButtonClicked() {
        bool ok;
        int roomNumber = QInputDialog::getInt(this, "Book Room", "Room Number:", 101, 1, 200, 1, &ok);
        if (!ok) return;

        if (roomExists(roomNumber)) {
            Room* room = searchRoom(roomsRoot, roomNumber);
            if (!room->isAvailable) {
                QMessageBox::warning(this, "Room Booking", "Room is already booked. Please choose another room.");
                return;
            }
        }

        QString roomType = QInputDialog::getText(this, "Add Room", "Room Type (Single/Double/Suite):", QLineEdit::Normal, "", &ok);
        if (!ok) return;
        double price = QInputDialog::getDouble(this, "Add Room", "Room Price:", 100.0, 0.0, 10000.0, 2, &ok);
        if (!ok) return;
        addRoom(roomNumber, roomType, price);
        qDebug() << "Room added successfully.";

        QString customerId = QInputDialog::getText(this, "Book Room", "Customer ID:", QLineEdit::Normal, "", &ok);
        if (!ok) return;
        QString customerName = QInputDialog::getText(this, "Book Room", "Customer Name:", QLineEdit::Normal, "", &ok);
        if (!ok) return;
        QString customerContact = QInputDialog::getText(this, "Book Room", "Customer Contact:", QLineEdit::Normal, "", &ok);
        if (!ok) return;

        addCustomer(customerName, customerId, customerContact);
        qDebug() << "Customer added successfully.";

        bookRoom(roomNumber, customerId);
    }

    void checkInButtonClicked() {
        bool ok;
        int roomNumber;
        QString customerId;
        QString checkInTime;

        do {
            roomNumber = QInputDialog::getInt(this, "Check In", "Room Number:", 101, 1, 200, 1, &ok);
            if (!ok) return;

            if (!roomExists(roomNumber)) {
                QMessageBox::warning(this, "Check In", "This room is not booked. Please enter a valid room number.");
                continue;
            }

            customerId = QInputDialog::getText(this, "Check In", "Customer ID:", QLineEdit::Normal, "", &ok);
            if (!ok) return;

            if (!customerExists(customerId)) {
                QMessageBox::warning(this, "Check In", "Customer ID not found. Please enter a valid customer ID.");
                continue;
            }

            checkInTime = QInputDialog::getText(this, "Check In", "Check-In Time (HH:MM):", QLineEdit::Normal, "", &ok);
            if (!ok) return;

            Room* room = searchRoom(roomsRoot, roomNumber);
            if (room != nullptr && !room->isAvailable && room->currentCustomerId == customerId) {
                room->checkInTime = checkInTime;
                QMessageBox::information(this, "Check In", "Check-in successful.");
                return;
            } else {
                QMessageBox::warning(this, "Check In", "Check-in failed. Room is either available or booked by another customer.");
            }
        } while (true);
    }

    void checkOutButtonClicked() {
        bool ok;
        int roomNumber;
        QString checkOutTime;

        do {
            roomNumber = QInputDialog::getInt(this, "Check Out", "Room Number:", 101, 1, 200, 1, &ok);
            if (!ok) return;

            if (!roomExists(roomNumber)) {
                QMessageBox::warning(this, "Check Out", "Invalid room number. Please enter a valid room number.");
                continue;
            }

            Room* room = searchRoom(roomsRoot, roomNumber);
            if (room != nullptr && !room->isAvailable && !room->checkInTime.isEmpty()) {
                checkOutTime = QInputDialog::getText(this, "Check Out", "Check-Out Time (HH:MM):", QLineEdit::Normal, "", &ok);
                if (!ok) return;

                room->isAvailable = true;
                room->checkOutTime = checkOutTime;

                // Remove the customer from the linked list
                removeCustomer(room->currentCustomerId);

                room->currentCustomerId = "";

                QMessageBox::information(this, "Check Out", "Check-out successful.");
                return;
            } else {
                QMessageBox::warning(this, "Check Out", "Check-out failed. Room is either already available or check-in was not done.");
            }
        } while (true);
    }
    void removeCustomer(const QString& customerId) {
        Customer* current = customersHead;
        Customer* previous = nullptr;

        while (current != nullptr && current->id != customerId) {
            previous = current;
            current = current->next;
        }

        if (current == nullptr) {
            // Customer not found
            return;
        }

        if (previous == nullptr) {
            // Customer is at the head
            customersHead = current->next;
        } else {
            previous->next = current->next;
        }

        delete current;
    }


    void displayAvailableRoomsButtonClicked() {
        QDialog dialog(this);
        dialog.setWindowTitle("Available Rooms");

        QVBoxLayout* dialogLayout = new QVBoxLayout(&dialog);

        displayAvailableRooms(roomsRoot, dialogLayout);

        QPushButton* closeButton = new QPushButton("Close", &dialog);
        dialogLayout->addWidget(closeButton);
        connect(closeButton, &QPushButton::clicked, &dialog, &QDialog::accept);

        dialog.exec();
    }

    void displayAvailableRooms(Room* root, QVBoxLayout* layout) {
        if (root != nullptr) {
            displayAvailableRooms(root->left, layout);
            if (root->isAvailable) {
                QLabel* roomLabel = new QLabel(
                    QString("Room %1: Type: %2, Price: %3")
                        .arg(root->roomNumber)
                        .arg(root->roomType)
                        .arg(root->price));
                layout->addWidget(roomLabel);
            }
            displayAvailableRooms(root->right, layout);
        }
    }

    void displayCustomersButtonClicked() {
        QDialog dialog(this);
        dialog.setWindowTitle("Customers");

        QVBoxLayout* dialogLayout = new QVBoxLayout(&dialog);

        displayCustomers(dialogLayout); // Root is not used in this function

        QPushButton* closeButton = new QPushButton("Close", &dialog);
        dialogLayout->addWidget(closeButton);
        connect(closeButton, &QPushButton::clicked, &dialog, &QDialog::accept);

        dialog.exec();
    }

    void displayBookingsButtonClicked() {
        QDialog dialog(this);
        dialog.setWindowTitle("Bookings");

        QVBoxLayout* dialogLayout = new QVBoxLayout(&dialog);

        displayBookings(roomsRoot, dialogLayout);

        QPushButton* closeButton = new QPushButton("Close", &dialog);
        dialogLayout->addWidget(closeButton);
        connect(closeButton, &QPushButton::clicked, &dialog, &QDialog::accept);

        dialog.exec();
    }

private:
    Room* searchRoom(Room* root, int roomNumber) {
        if (root == nullptr || root->roomNumber == roomNumber) {
            return root;
        } else if (roomNumber < root->roomNumber) {
            return searchRoom(root->left, roomNumber);
        } else {
            return searchRoom(root->right, roomNumber);
        }
    }

    void addRoom(int roomNumber, const QString& roomType, double price) {
        roomsRoot = insertRoom(roomsRoot, roomNumber, roomType, price);
    }

    Room* insertRoom(Room* root, int roomNumber, const QString& roomType, double price) {
        if (root == nullptr) {
            return new Room(roomNumber, roomType, price);
        }

        if (roomNumber < root->roomNumber) {
            root->left = insertRoom(root->left, roomNumber, roomType, price);
        } else if (roomNumber > root->roomNumber) {
            root->right = insertRoom(root->right, roomNumber, roomType, price);
        }

        return root;
    }

    bool roomExists(int roomNumber) {
        return searchRoom(roomsRoot, roomNumber) != nullptr;
    }

    Customer* findCustomer(const QString& id) {
        Customer* current = customersHead;
        while (current != nullptr) {
            if (current->id == id) {
                return current;
            }
            current = current->next;
        }
        return nullptr;
    }

    void addCustomer(const QString& name, const QString& id, const QString& contact) {
        Customer* existingCustomer = findCustomer(id);
        if (existingCustomer) {
            // If customer exists, update the information
            existingCustomer->name = name;
            existingCustomer->contact = contact;
            return;
        }

        Customer* newCustomer = new Customer(name, id, contact);
        newCustomer->next = customersHead;
        customersHead = newCustomer;
    }

    bool customerExists(const QString& id) {
        return findCustomer(id) != nullptr;
    }

    void displayCustomers(QVBoxLayout* layout) {
        customersHead = mergeSort(customersHead); // Sort the linked list

        Customer* current = customersHead;
        while (current != nullptr) {
            QLabel* customerLabel = new QLabel(
                QString("Name: %1, ID: %2, Contact: %3")
                    .arg(current->name)
                    .arg(current->id)
                    .arg(current->contact));
            layout->addWidget(customerLabel);
            current = current->next;
        }
    }

    // Merge Sort for linked list
    Customer* merge(Customer* left, Customer* right) {
        if (!left)
            return right;
        if (!right)
            return left;

        Customer* result = nullptr;

        if (left->name <= right->name) {
            result = left;
            result->next = merge(left->next, right);
        } else {
            result = right;
            result->next = merge(left, right->next);
        }

        return result;
    }

    Customer* mergeSort(Customer* head) {
        if (!head || !head->next)
            return head;

        Customer* slow = head;
        Customer* fast = head->next;

        while (fast && fast->next) {
            slow = slow->next;
            fast = fast->next->next;
        }

        Customer* mid = slow->next;
        slow->next = nullptr;

        return merge(mergeSort(head), mergeSort(mid));
    }



    void displayBookings(Room* root, QVBoxLayout* layout) {
        QMap<QString, QStringList> bookingsByCustomerId;

        // Traverse the tree to collect bookings by customer ID
        collectBookings(root, bookingsByCustomerId);

        // Display bookings grouped by customer ID
        for (const QString& customerId : bookingsByCustomerId.keys()) {
            QStringList roomBookings = bookingsByCustomerId[customerId];
            QLabel* customerLabel = new QLabel(QString("Customer ID: %1").arg(customerId));
            layout->addWidget(customerLabel);
            for (const QString& booking : roomBookings) {
                QLabel* bookingLabel = new QLabel(booking);
                layout->addWidget(bookingLabel);
            }
        }
    }

    void collectBookings(Room* root, QMap<QString, QStringList>& bookingsByCustomerId) {
        if (root != nullptr) {
            collectBookings(root->left, bookingsByCustomerId);
            if (!root->isAvailable) {
                QString bookingInfo = QString("Room %1: Check-In: %2, Check-Out: %3")
                                          .arg(root->roomNumber)
                                          .arg(root->checkInTime)
                                          .arg(root->checkOutTime);
                bookingsByCustomerId[root->currentCustomerId].append(bookingInfo);
            }
            collectBookings(root->right, bookingsByCustomerId);
        }
    }


    void checkIn(int roomNumber, const QString& customerId, const QString& checkInTime) {
        Room* room = searchRoom(roomsRoot, roomNumber);
        if (room != nullptr && room->isAvailable) {
            room->isAvailable = false;
            room->currentCustomerId = customerId;
            room->checkInTime = checkInTime;
            room->checkOutTime = "";
            QMessageBox::information(this, "Check In", "Check-in successful.");
        } else {
            QMessageBox::warning(this, "Check In", "Check-in failed. Room is not available.");
        }
    }

    void checkOut(int roomNumber, const QString& checkOutTime) {
        Room* room = searchRoom(roomsRoot, roomNumber);
        if (room != nullptr && !room->isAvailable) {
            room->isAvailable = true;
            room->checkOutTime = checkOutTime;
            QMessageBox::information(this, "Check Out", "Check-out successful.");
        } else {
            QMessageBox::warning(this, "Check Out", "Check-out failed. Room is already available.");
        }
    }

    void bookRoom(int roomNumber, const QString& customerId) {
        Room* room = searchRoom(roomsRoot, roomNumber);
        if (room != nullptr) {
            room->isAvailable = false;
            room->currentCustomerId = customerId;
        }
    }
    void displayRoomInfo(int roomNumber) {
        Room* room = searchRoom(roomsRoot, roomNumber);
        if (room != nullptr) {
            QString availability = room->isAvailable ? "Available" : "Not Available";
            QString info = QString("Room %1: Type: %2, Price: %3, Availability: %4")
                               .arg(room->roomNumber)
                               .arg(room->roomType)
                               .arg(room->price)
                               .arg(availability);
            if (!room->isAvailable) {
                info += QString(", Current Customer ID: %1, Check-In Time: %2, Check-Out Time: %3")
                            .arg(room->currentCustomerId)
                            .arg(room->checkInTime)
                            .arg(room->checkOutTime);
            }
            QMessageBox::information(this, "Room Info", info);
        } else {
            QMessageBox::warning(this, "Room Info", "Room not found.");
        }
    }

    void search() {
        QString searchText = searchBar->text();
        QString option = searchOptions->currentText();

        if (option == "Room Number") {
            bool ok;
            int roomNumber = searchText.toInt(&ok);
            if (ok) {
                displayRoomInfo(roomNumber);
            } else {
                QMessageBox::warning(this, "Search", "Invalid room number.");
            }
        } else if (option == "Customer ID") {
            Customer* customer = findCustomer(searchText);
            if (customer != nullptr) {
                QString info = QString("Name: %1, ID: %2, Contact: %3")
                                   .arg(customer->name)
                                   .arg(customer->id)
                                   .arg(customer->contact);
                QMessageBox::information(this, "Customer Info", info);
            } else {
                QMessageBox::warning(this, "Search", "Customer not found.");
            }
        }
    }
};

int main(int argc, char *argv[]) {
    QApplication a(argc, argv);

    HotelManagementSystem hms;
    hms.show();

    return a.exec();
}
#include "main.moc"
