#define WINVER 0x0601
#include <iostream>
#include <fstream>
#include <string>
#include <windows.h>
#include <iomanip>
using namespace std;

// Global arrays (max 100 students)
int roll[100];
string name[100];
float gpa[100];
char grade[100];
int marks[100][5];
int totalStudents = 0;

// Subject names
string subjects[5] = {"Calculus", "EDCD", "English", "Statics", "Physics"};

// Serial handle
HANDLE com;

// ---------- Serial functions ----------
bool openPort(string port) {
    string path = "\\\\.\\" + port;
    com = CreateFileA(path.c_str(), GENERIC_READ|GENERIC_WRITE, 0, NULL, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, NULL);
    if (com == INVALID_HANDLE_VALUE) return false;
    DCB dcb = {0};
    dcb.DCBlength = sizeof(dcb);
    GetCommState(com, &dcb);
    dcb.BaudRate = 115200;
    dcb.ByteSize = 8;
    dcb.StopBits = ONESTOPBIT;
    dcb.Parity = NOPARITY;
    SetCommState(com, &dcb);
    return true;
}

bool authenticate() {
    DWORD written;
    WriteFile(com, "GET_KEY\n", 8, &written, NULL);
    char buf[100];
    DWORD read;
    string key = "";
    while (true) {
        ReadFile(com, buf, 1, &read, NULL);
        if (read == 0) break;
        if (buf[0] == '\n') break;
        if (buf[0] != '\r') key += buf[0];
    }
    // read and discard the roll number
    while (true) {
        ReadFile(com, buf, 1, &read, NULL);
        if (read == 0) break;
        if (buf[0] == '\n') break;
    }
    return (key == "ESP_KEY_SG_001");
}

// ---------- CSV file functions ----------
void loadData() {
    ifstream file("students.csv");
    if (!file) {
        cout << "students.csv not found!\n";
        exit(1);
    }
    string line;
    getline(file, line); // skip header
    totalStudents = 0;
    while (getline(file, line) && totalStudents < 100) {
        // parse line: roll,name,gpa,grade,m1,m2,m3,m4,m5
        int pos = 0, comma;
        comma = line.find(',');
        roll[totalStudents] = stoi(line.substr(0, comma));
        pos = comma + 1;
        comma = line.find(',', pos);
        name[totalStudents] = line.substr(pos, comma - pos);
        pos = comma + 1;
        comma = line.find(',', pos);
        gpa[totalStudents] = stof(line.substr(pos, comma - pos));
        pos = comma + 1;
        comma = line.find(',', pos);
        grade[totalStudents] = line.substr(pos, comma - pos)[0];
        for (int i = 0; i < 5; i++) {
            pos = comma + 1;
            comma = line.find(',', pos);
            if (comma == -1) comma = line.length();
            marks[totalStudents][i] = stoi(line.substr(pos, comma - pos));
        }
        totalStudents++;
    }
    file.close();
    cout << "Loaded " << totalStudents << " students.\n";
}

void saveData() {
    ofstream file("students.csv");
    file << "Roll,Name,GPA,Grade,Calculus,EDCD,English,Statics,Physics\n";
    for (int i = 0; i < totalStudents; i++) {
        file << roll[i] << "," << name[i] << "," << gpa[i] << "," << grade[i];
        for (int j = 0; j < 5; j++) file << "," << marks[i][j];
        file << "\n";
    }
    file.close();
}

// ---------- Helper functions ----------
void calculateGPAgrade(int idx) {
    int total = 0;
    for (int i = 0; i < 5; i++) total += marks[idx][i];
    gpa[idx] = (total / 500.0) * 4.0;
    if (gpa[idx] >= 3.5) grade[idx] = 'A';
    else if (gpa[idx] >= 3.0) grade[idx] = 'B';
    else if (gpa[idx] >= 2.5) grade[idx] = 'C';
    else if (gpa[idx] >= 2.0) grade[idx] = 'D';
    else grade[idx] = 'F';
}

void displayStudent(int idx) {
    cout << "\n========================================\n";
    cout << "Roll: " << roll[idx] << " | Name: " << name[idx] << " | GPA: " << gpa[idx] << " | Grade: " << grade[idx] << "\n";
    cout << "----------------------------------------\n";
    cout << left << setw(12) << "Subject" << " | Marks\n";
    cout << "----------------------------------------\n";
    for (int i = 0; i < 5; i++) {
        cout << left << setw(12) << subjects[i] << " | " << marks[idx][i] << "\n";
    }
    cout << "========================================\n";
}
// ---------- Main operations ----------
int searchByRoll(int r) {
    for (int i = 0; i < totalStudents; i++)
        if (roll[i] == r) return i;
    return -1;
}

void addStudent() {
    if (totalStudents >= 100) { cout << "Database full.\n"; return; }
    int r;
    cout << "Enter Roll Number: "; cin >> r;
    if (searchByRoll(r) != -1) { cout << "Roll already exists.\n"; return; }
    roll[totalStudents] = r;
    cout << "Enter Name: "; cin.ignore(); getline(cin, name[totalStudents]);
    cout << "Enter marks (0-100):\n";
    for (int i = 0; i < 5; i++) {
        do {
            cout << subjects[i] << ": "; cin >> marks[totalStudents][i];
        } while (marks[totalStudents][i] < 0 || marks[totalStudents][i] > 100);
    }
    calculateGPAgrade(totalStudents);
    totalStudents++;
    saveData();
    cout << "Student added.\n";
}

void updateStudent() {
    int r;
    cout << "Enter Roll Number to update: "; cin >> r;
    int idx = searchByRoll(r);
    if (idx == -1) { cout << "Not found.\n"; return; }
    cout << "Enter new Name (current: " << name[idx] << "): ";
    cin.ignore(); getline(cin, name[idx]);
    cout << "Enter new marks:\n";
    for (int i = 0; i < 5; i++) {
        do {
            cout << subjects[i] << " (current: " << marks[idx][i] << "): ";
            cin >> marks[idx][i];
        } while (marks[idx][i] < 0 || marks[idx][i] > 100);
    }
    calculateGPAgrade(idx);
    saveData();
    cout << "Updated.\n";
}

void deleteStudent() {
    int r;
    cout << "Enter Roll Number to delete: "; cin >> r;
    int idx = searchByRoll(r);
    if (idx == -1) { cout << "Not found.\n"; return; }
    for (int i = idx; i < totalStudents - 1; i++) {
        roll[i] = roll[i+1];
        name[i] = name[i+1];
        gpa[i] = gpa[i+1];
        grade[i] = grade[i+1];
        for (int j = 0; j < 5; j++) marks[i][j] = marks[i+1][j];
    }
    totalStudents--;
    saveData();
    cout << "Deleted.\n";
}

void listAll() {
    cout << "\n--- Student List ---\n";
    for (int i = 0; i < totalStudents; i++)
        cout << "Roll: " << roll[i] << " | " << name[i] << "\n";
}

// ---------- Main ----------
int main() {
    loadData();

    string comPort;
    cout << "Enter COM port (e.g. COM4): "; cin >> comPort;
    if (!openPort(comPort)) {
        cout << "Cannot open " << comPort << ". Check cable.\n";
        return 1;
    }
    if (!authenticate()) {
        cout << "ESP32 key invalid.\n";
        return 1;
    }
    cout << "ESP32 OK.\n";

    int choice;
    do {
        cout << "\n1 Search  2 Delete  3 Add  4 Update  5 List  6 Exit\nChoice: ";
        cin >> choice;
        if (choice == 1) {
            int r; cout << "Roll: "; cin >> r;
            int idx = searchByRoll(r);
            if (idx == -1) cout << "Not found.\n";
            else displayStudent(idx);
        }
        else if (choice == 2) deleteStudent();
        else if (choice == 3) addStudent();
        else if (choice == 4) updateStudent();
        else if (choice == 5) listAll();
    } while (choice != 6);

    CloseHandle(com);
    return 0;
}

