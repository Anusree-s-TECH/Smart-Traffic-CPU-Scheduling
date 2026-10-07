# Smart-Traffic-CPU-Scheduling
Priority Preemptive CPU Scheduling for a Smart Traffic Control Center using ESP32
// =====================================================
// SMART TRAFFIC CONTROL CENTER
// PRIORITY PREEMPTIVE CPU SCHEDULING
// ESP32 + 4 LEDs
// =====================================================

// LED PIN CONNECTIONS
#define LED_T1 13   // P1 - Traffic Monitoring
#define LED_T2 12   // P2 - Congestion Alert
#define LED_T3 14   // P3 - Signal Failure
#define LED_T4 27   // P4 - Accident Alert

// -----------------------------------------------------
// PROCESS STRUCTURE
// -----------------------------------------------------

struct Process {
  String id;
  String event;

  int arrivalTime;
  int burstTime;
  int remainingTime;
  int priority;

  int completionTime;
  int waitingTime;
  int turnaroundTime;
};

// -----------------------------------------------------
// PROCESS DETAILS
// Lower priority number = Higher priority
// -----------------------------------------------------

Process p[4] = {

  // ID   Event                AT   BT   Remaining   Priority
  {"P1", "Traffic Monitoring", 0,   5,   5,           4},
  {"P2", "Congestion Alert",   1,   3,   3,           3},
  {"P3", "Signal Failure",     2,   4,   4,           2},
  {"P4", "Accident Alert",     4,   2,   2,           1}
};

// LED pins for each process
int ledPins[4] = {
  LED_T1,
  LED_T2,
  LED_T3,
  LED_T4
};

// -----------------------------------------------------
// VARIABLES FOR GANTT CHART
// -----------------------------------------------------

int ganttProcess[20];
int ganttStart[20];
int ganttEnd[20];
int ganttCount = 0;

// -----------------------------------------------------
// TURN OFF ALL LEDs
// -----------------------------------------------------

void allLEDsOff() {

  digitalWrite(LED_T1, LOW);
  digitalWrite(LED_T2, LOW);
  digitalWrite(LED_T3, LOW);
  digitalWrite(LED_T4, LOW);
}

// -----------------------------------------------------
// TURN ON LED FOR SELECTED PROCESS
// -----------------------------------------------------

void runLED(int processNumber) {

  allLEDsOff();

  digitalWrite(
    ledPins[processNumber],
    HIGH
  );
}

// -----------------------------------------------------
// FIND HIGHEST PRIORITY READY PROCESS
// -----------------------------------------------------

int selectProcess(int currentTime) {

  int selected = -1;

  for (int i = 0; i < 4; i++) {

    // Process must have arrived
    // and must still have execution time
    if (p[i].arrivalTime <= currentTime &&
        p[i].remainingTime > 0) {

      // Smaller priority number = higher priority
      if (selected == -1 ||
          p[i].priority < p[selected].priority) {

        selected = i;
      }
    }
  }

  return selected;
}

// -----------------------------------------------------
// DISPLAY PROCESS INFORMATION
// -----------------------------------------------------

void displayProcessTable() {

  Serial.println();
  Serial.println("==============================================================");
  Serial.println("PROCESS INFORMATION");
  Serial.println("==============================================================");

  Serial.println(
    "Process\tEvent\t\t\tAT\tBT\tPriority"
  );

  Serial.println(
    "--------------------------------------------------------------"
  );

  for (int i = 0; i < 4; i++) {

    Serial.print(p[i].id);
    Serial.print("\t");

    Serial.print(p[i].event);

    // Formatting
    if (p[i].event.length() < 16)
      Serial.print("\t");

    Serial.print("\t");

    Serial.print(p[i].arrivalTime);
    Serial.print("\t");

    Serial.print(p[i].burstTime);
    Serial.print("\t");

    Serial.println(p[i].priority);
  }

  Serial.println(
    "=============================================================="
  );

  Serial.println();
  Serial.println(
    "NOTE: Lower priority number = Higher priority"
  );

  Serial.println();
}

// -----------------------------------------------------
// PRIORITY PREEMPTIVE SCHEDULER
// -----------------------------------------------------

void priorityPreemptiveScheduling() {

  int currentTime = 0;
  int completed = 0;

  int previousProcess = -1;

  Serial.println();
  Serial.println("==============================================================");
  Serial.println("       PRIORITY PREEMPTIVE EXECUTION STARTED");
  Serial.println("==============================================================");
  Serial.println();

  // Continue until all processes complete
  while (completed < 4) {

    // Select highest priority process
    int selected = selectProcess(currentTime);

    // -------------------------------------------------
    // CPU IDLE
    // -------------------------------------------------

    if (selected == -1) {

      Serial.print("Time ");
      Serial.print(currentTime);
      Serial.println(" -> CPU IDLE");

      currentTime++;

      delay(1000);

      continue;
    }

    // -------------------------------------------------
    // CHECK FOR PREEMPTION
    // -------------------------------------------------

    if (previousProcess != -1 &&
        previousProcess != selected &&
        p[previousProcess].remainingTime > 0) {

      Serial.println();
      Serial.print("*** PREEMPTION ***  ");
      Serial.print(p[previousProcess].id);
      Serial.print(" is PREEMPTED by ");
      Serial.println(p[selected].id);

      Serial.print("Reason: ");
      Serial.print(p[selected].event);
      Serial.println(" has higher priority.");

      Serial.println();
    }

    // -------------------------------------------------
    // GANTT CHART START
    // -------------------------------------------------

    if (ganttCount == 0 ||
        ganttProcess[ganttCount - 1] != selected) {

      if (ganttCount > 0) {
        ganttEnd[ganttCount - 1] = currentTime;
      }

      ganttProcess[ganttCount] = selected;
      ganttStart[ganttCount] = currentTime;

      ganttCount++;
    }

    // -------------------------------------------------
    // RUN PROCESS
    // -------------------------------------------------

    runLED(selected);

    Serial.print("Time ");
    Serial.print(currentTime);
    Serial.print(" -> ");
    Serial.print(p[selected].id);
    Serial.print(" : ");
    Serial.print(p[selected].event);
    Serial.print(" | Priority = ");
    Serial.println(p[selected].priority);

    // Execute for ONE time unit
    p[selected].remainingTime--;

    previousProcess = selected;

    currentTime++;

    delay(1000);

    // -------------------------------------------------
    // PROCESS COMPLETED
    // -------------------------------------------------

    if (p[selected].remainingTime == 0) {

      p[selected].completionTime = currentTime;

      p[selected].turnaroundTime =
        p[selected].completionTime -
        p[selected].arrivalTime;

      p[selected].waitingTime =
        p[selected].turnaroundTime -
        p[selected].burstTime;

      completed++;

      Serial.println();

      Serial.print(">>> ");
      Serial.print(p[selected].id);
      Serial.print(" COMPLETED at Time ");
      Serial.println(currentTime);

      Serial.print("Waiting Time = ");
      Serial.println(p[selected].waitingTime);

      Serial.print("Turnaround Time = ");
      Serial.println(p[selected].turnaroundTime);

      Serial.println();
    }
  }

  // Close last Gantt block
  if (ganttCount > 0) {
    ganttEnd[ganttCount - 1] = currentTime;
  }

  allLEDsOff();

  displayGanttChart();

  displayFinalResults();
}

// -----------------------------------------------------
// DISPLAY GANTT CHART
// -----------------------------------------------------

void displayGanttChart() {

  Serial.println();
  Serial.println("==============================================================");
  Serial.println("                    GANTT CHART");
  Serial.println("==============================================================");

  Serial.println();

  // Process names
  for (int i = 0; i < ganttCount; i++) {

    Serial.print("|   ");
    Serial.print(p[ganttProcess[i]].id);
    Serial.print("   ");
  }

  Serial.println("|");

  // Time values
  Serial.print(ganttStart[0]);

  for (int i = 0; i < ganttCount; i++) {

    Serial.print("\t");
    Serial.print(ganttEnd[i]);
  }

  Serial.println();

  Serial.println();
  Serial.println("Execution Sequence:");

  for (int i = 0; i < ganttCount; i++) {

    Serial.print(p[ganttProcess[i]].id);

    if (i < ganttCount - 1) {
      Serial.print(" -> ");
    }
  }

  Serial.println();

  Serial.println();
}

// -----------------------------------------------------
// DISPLAY FINAL RESULTS
// -----------------------------------------------------

void displayFinalResults() {

  float totalWaiting = 0;
  float totalTurnaround = 0;

  Serial.println("==============================================================");
  Serial.println("                    FINAL RESULTS");
  Serial.println("==============================================================");

  Serial.println();

  Serial.println(
    "Process\tAT\tBT\tPriority\tCT\tWT\tTAT"
  );

  Serial.println(
    "--------------------------------------------------------------"
  );

  for (int i = 0; i < 4; i++) {

    Serial.print(p[i].id);
    Serial.print("\t");

    Serial.print(p[i].arrivalTime);
    Serial.print("\t");

    Serial.print(p[i].burstTime);
    Serial.print("\t");

    Serial.print(p[i].priority);
    Serial.print("\t\t");

    Serial.print(p[i].completionTime);
    Serial.print("\t");

    Serial.print(p[i].waitingTime);
    Serial.print("\t");

    Serial.println(p[i].turnaroundTime);

    totalWaiting += p[i].waitingTime;
    totalTurnaround += p[i].turnaroundTime;
  }

  Serial.println(
    "--------------------------------------------------------------"
  );

  float averageWaiting =
    totalWaiting / 4.0;

  float averageTurnaround =
    totalTurnaround / 4.0;

  Serial.println();

  Serial.print("Average Waiting Time    = ");
  Serial.println(averageWaiting);

  Serial.print("Average Turnaround Time = ");
  Serial.println(averageTurnaround);

  Serial.println();

  Serial.println("==============================================================");
  Serial.println("             SCHEDULING COMPLETED");
  Serial.println("==============================================================");
}

// -----------------------------------------------------
// SETUP
// -----------------------------------------------------

void setup() {

  // LED pins
  pinMode(LED_T1, OUTPUT);
  pinMode(LED_T2, OUTPUT);
  pinMode(LED_T3, OUTPUT);
  pinMode(LED_T4, OUTPUT);

  allLEDsOff();

  // Serial Monitor
  Serial.begin(115200);

  delay(1000);

  Serial.println();
  Serial.println("==============================================================");
  Serial.println("             SMART TRAFFIC CONTROL CENTER");
  Serial.println("==============================================================");

  Serial.println();
  Serial.println("Scheduling Algorithm: PRIORITY PREEMPTIVE");
  Serial.println();

  displayProcessTable();

  delay(2000);

  priorityPreemptiveScheduling();
}

// -----------------------------------------------------
// LOOP
// -----------------------------------------------------

void loop() {

  // Scheduling is executed only once.
}
