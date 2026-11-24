#include <iostream>
#include <string>
#include <stdexcept>
#include <cstdlib>
#include <ctime>
#include <cctype>
#include <iomanip>

using namespace std;

// ============================================================
//  Constants
// ============================================================
const int NUM_OPTIONS         = 4;    // answers A, B, C, D
const int QUESTIONS_PER_ROUND = 10;
const int POINTS_PER_QUESTION = 10;
const int STREAK_FOR_BONUS    = 3;    // bonus every 3 correct in a row
const int STREAK_BONUS        = 5;
const int MAX_LEADERS         = 10;   // top-10 leaderboard
const int MAX_NAME_LENGTH     = 20;
const int HUD_WIDTH           = 58;   // inside width of the HUD box

// Screen clearing and colors use ANSI codes, which work in OnlineGDB and
// most terminals. If you see strange symbols like [2J, set this to false.
const bool USE_ANSI = true;

const int NUM_CATEGORIES = 10;
const string CATEGORIES[NUM_CATEGORIES] = {
    "Science", "History", "Geography", "Movies and TV", "Sports",
    "Basic Math", "Literature", "English", "Art", "Music"
};

// ============================================================
//  Custom exception classes
// ============================================================

// Thrown when the user types something the program can't accept.
class InvalidInputException : public runtime_error {
public:
    explicit InvalidInputException(const string& message)
        : runtime_error(message) {}
};


// ============================================================
//  Helper functions (text and display)
// ============================================================

// Removes spaces and tabs from the start and end of a string.
string trim(const string& text) {
    size_t start = text.find_first_not_of(" \t\r\n");
    if (start == string::npos) {
        return "";
    }
    size_t end = text.find_last_not_of(" \t\r\n");
    return text.substr(start, end - start + 1);
}

// Returns true if the string contains only digits (and is not empty).
bool isWholeNumber(const string& text) {
    if (text.empty()) {
        return false;
    }
    for (size_t i = 0; i < text.length(); i++) {
        if (!isdigit(static_cast<unsigned char>(text[i]))) {
            return false;
        }
    }
    return true;
}

// Prints a line of repeated characters, used to separate sections.
void printLine(char symbol = '=', int width = 60) {
    for (int i = 0; i < width; i++) {
        cout << symbol;
    }
    cout << endl;
}

// Prints a title between two lines.
void printTitle(const string& title) {
    cout << endl;
    printLine('=');
    cout << "  " << title << endl;
    printLine('=');
}

// Clears the screen and moves the cursor to the top-left corner.
void clearScreen() {
    if (USE_ANSI) {
        cout << "\033[2J\033[H";
    } else {
        cout << endl << endl;
    }
}

// Text colors for feedback (green = correct, red = wrong, yellow = bonus).
string green(const string& text) {
    return USE_ANSI ? "\033[32m" + text + "\033[0m" : text;
}
string red(const string& text) {
    return USE_ANSI ? "\033[31m" + text + "\033[0m" : text;
}
string yellow(const string& text) {
    return USE_ANSI ? "\033[33m" + text + "\033[0m" : text;
}

// Prints the top or bottom border of a box: +-----...-----+
void printBoxBorder() {
    cout << "+";
    for (int i = 0; i < HUD_WIDTH; i++) {
        cout << "-";
    }
    cout << "+" << endl;
}

// Prints one row inside a box, padded with spaces so the right edge lines up.
void printBoxRow(const string& content) {
    string row = " " + content;
    if (row.length() > static_cast<size_t>(HUD_WIDTH)) {
        row = row.substr(0, HUD_WIDTH);   // cut off anything too long
    }
    cout << "|" << row;
    for (size_t i = row.length(); i < static_cast<size_t>(HUD_WIDTH); i++) {
        cout << " ";
    }
    cout << "|" << endl;
}

// Returns text padded with spaces on the right to a fixed width.
string padRight(const string& text, size_t width) {
    if (text.length() >= width) {
        return text;
    }
    return text + string(width - text.length(), ' ');
}

// Reads one full line from the keyboard. If input ends (for example,
// the window is closed), the program exits cleanly.
string readLine() {
    string line;
    if (!getline(cin, line)) {
        cout << endl << "Input ended. Goodbye!" << endl;
        exit(0);
    }
    return trim(line);
}

// Waits until the user presses Enter.
void pressEnterToContinue() {
    cout << endl << "Press Enter to continue...";
    readLine();
}

// ============================================================
//  Input functions (these throw InvalidInputException)
// ============================================================

// Reads a whole number between minValue and maxValue.
int readInt(const string& prompt, int minValue, int maxValue) {
    cout << prompt;
    string input = readLine();

    if (!isWholeNumber(input)) {
        throw InvalidInputException("\"" + input + "\" is not a whole number.");
    }
    if (input.length() > 6) {
        throw InvalidInputException("That number is too large.");
    }

    int value = stoi(input);
    if (value < minValue || value > maxValue) {
        throw InvalidInputException("Please enter a number from " +
                                    to_string(minValue) + " to " +
                                    to_string(maxValue) + ".");
    }
    return value;
}

// Reads an answer letter (A, B, C or D) and returns it in uppercase.
char readAnswer(const string& prompt) {
    cout << prompt;
    string input = readLine();

    if (input.length() != 1) {
        throw InvalidInputException("Type a single letter: A, B, C or D.");
    }
    char letter = static_cast<char>(toupper(static_cast<unsigned char>(input[0])));
    if (letter < 'A' || letter > 'D') {
        throw InvalidInputException("\"" + input + "\" is not A, B, C or D.");
    }
    return letter;
}

// Reads a name: it can't be empty or too long.
string readName(const string& prompt) {
    cout << prompt;
    string input = readLine();

    if (input.empty()) {
        throw InvalidInputException("The name can't be empty.");
    }
    if (input.length() > static_cast<size_t>(MAX_NAME_LENGTH)) {
        throw InvalidInputException("Use at most " + to_string(MAX_NAME_LENGTH) +
                                    " characters.");
    }
    return input;
}

// The "ask" functions repeat the question until the input is valid.
// They catch the exception, show the error, and ask again.
int askInt(const string& prompt, int minValue, int maxValue) {
    while (true) {
        try {
            return readInt(prompt, minValue, maxValue);
        } catch (const InvalidInputException& e) {
            cout << "  Invalid input: " << e.what() << " Try again." << endl;
        }
    }
}

char askAnswer(const string& prompt) {
    while (true) {
        try {
            return readAnswer(prompt);
        } catch (const InvalidInputException& e) {
            cout << "  Invalid input: " << e.what() << " Try again." << endl;
        }
    }
}

string askName(const string& prompt) {
    while (true) {
        try {
            return readName(prompt);
        } catch (const InvalidInputException& e) {
            cout << "  Invalid input: " << e.what() << " Try again." << endl;
        }
    }
}

// ============================================================
//  Class: Question
//  Stores one multiple-choice question and checks answers.
// ============================================================
class Question {
private:
    string text;
    string options[NUM_OPTIONS];   // array of the four answer choices
    int correctIndex;              // 0 = A, 1 = B, 2 = C, 3 = D
    int category;                  // index into the CATEGORIES array

public:
    // Default constructor (needed to create arrays of Question).
    Question() : text(""), correctIndex(0), category(0) {}

    // Constructor that fills in every field.
    Question(int questionCategory, const string& questionText,
             const string& a, const string& b, const string& c, const string& d,
             char answer)
        : text(questionText), category(questionCategory) {
        options[0] = a;
        options[1] = b;
        options[2] = c;
        options[3] = d;
        correctIndex = toupper(static_cast<unsigned char>(answer)) - 'A';
    }

    // Getters
    string getText() const     { return text; }
    int getCategory() const    { return category; }
    string getCategoryName() const { return CATEGORIES[category]; }

    char getCorrectLetter() const {
        return static_cast<char>('A' + correctIndex);
    }

    string getCorrectOptionText() const {
        return options[correctIndex];
    }

    string getOptionText(char letter) const {
        return options[letter - 'A'];
    }

    bool isCorrect(char answer) const {
        return toupper(static_cast<unsigned char>(answer)) - 'A' == correctIndex;
    }

    // Shuffles the four options so the right answer moves to a new
    // letter each game. The correct index is updated as it moves.
    void shuffleOptions() {
        for (int i = NUM_OPTIONS - 1; i > 0; i--) {
            int j = rand() % (i + 1);

            string temp = options[i];
            options[i] = options[j];
            options[j] = temp;

            // Keep track of where the right answer went.
            if (correctIndex == i) {
                correctIndex = j;
            } else if (correctIndex == j) {
                correctIndex = i;
            }
        }
    }

    // Shows the question and its four options under the HUD.
    void display() const {
        cout << endl << "  " << text << endl << endl;
        for (int i = 0; i < NUM_OPTIONS; i++) {
            cout << "     " << static_cast<char>('A' + i) << ") " << options[i] << endl;
        }
    }
};

// ============================================================
//  Class: Player
//  Tracks the player's name, points, streak and results.
// ============================================================
class Player {
private:
    string name;
    int points;
    int correctAnswers;
    int questionsAnswered;
    int streak;                                   // correct in a row right now
    int bestStreak;
    int correctByCategory[NUM_CATEGORIES];        // correct answers per category
    int askedByCategory[NUM_CATEGORIES];          // questions asked per category

public:
    explicit Player(const string& playerName)
        : name(playerName), points(0), correctAnswers(0), questionsAnswered(0),
          streak(0), bestStreak(0) {
        for (int i = 0; i < NUM_CATEGORIES; i++) {
            correctByCategory[i] = 0;
            askedByCategory[i] = 0;
        }
    }

    // Records a correct answer. Returns the streak bonus earned (0 if none).
    int recordCorrect(int category) {
        questionsAnswered++;
        correctAnswers++;
        askedByCategory[category]++;
        correctByCategory[category]++;
        points += POINTS_PER_QUESTION;

        streak++;
        if (streak > bestStreak) {
            bestStreak = streak;
        }

        int bonus = 0;
        if (streak % STREAK_FOR_BONUS == 0) {
            bonus = STREAK_BONUS;
            points += bonus;
        }
        return bonus;
    }

    // Records a wrong answer and resets the streak.
    void recordWrong(int category) {
        questionsAnswered++;
        askedByCategory[category]++;
        streak = 0;
    }

    // Getters
    string getName() const           { return name; }
    int getPoints() const            { return points; }
    int getCorrectAnswers() const    { return correctAnswers; }
    int getQuestionsAnswered() const { return questionsAnswered; }
    int getStreak() const            { return streak; }
    int getBestStreak() const        { return bestStreak; }

    double getPercentage() const {
        if (questionsAnswered == 0) {
            return 0.0;
        }
        return 100.0 * correctAnswers / questionsAnswered;
    }

    // Letter grade based on the percentage of correct answers.
    char getGrade() const {
        double percent = getPercentage();
        if (percent >= 90) return 'A';
        if (percent >= 80) return 'B';
        if (percent >= 70) return 'C';
        if (percent >= 60) return 'D';
        return 'F';
    }

    // Draws the HUD (heads-up display) box at the top of the screen.
    void displayHUD(int questionNumber, int totalQuestions,
                    const string& categoryName) const {
        string streakText = "Streak: " + to_string(streak);
        int toBonus = STREAK_FOR_BONUS - (streak % STREAK_FOR_BONUS);
        if (streak > 0 && toBonus == 1) {
            streakText += " (1 more: +" + to_string(STREAK_BONUS) + ")";
        }

        printBoxBorder();
        printBoxRow(padRight("Player: " + name, 37) + "Question " +
                    to_string(questionNumber) + " of " + to_string(totalQuestions));
        printBoxRow(padRight("Points: " + to_string(points), 14) +
                    padRight("Correct: " + to_string(correctAnswers) + "/" +
                             to_string(questionsAnswered), 16) + streakText);
        printBoxRow("Category: " + categoryName);
        printBoxBorder();
    }

    // Shows how the player did in each category that came up.
    void displayCategoryBreakdown() const {
        cout << endl << "  Results by category:" << endl;
        for (int i = 0; i < NUM_CATEGORIES; i++) {
            if (askedByCategory[i] > 0) {
                cout << "    " << left << setw(16) << CATEGORIES[i]
                     << correctByCategory[i] << " of " << askedByCategory[i]
                     << " correct" << endl;
            }
        }
    }
};

// ============================================================
//  Class: QuestionBank
//  Holds all 100 questions in a dynamic array (created with new).
// ============================================================
class QuestionBank {
private:
    Question* questions;   // pointer to a dynamic array of questions
    int count;             // how many questions are stored
    int capacity;          // how many fit before the array must grow

    // Doubles the size of the array when it is full.
    void grow() {
        int newCapacity = capacity * 2;
        Question* bigger = new Question[newCapacity];
        for (int i = 0; i < count; i++) {
            bigger[i] = questions[i];
        }
        delete[] questions;      // free the old array
        questions = bigger;      // point to the new one
        capacity = newCapacity;
    }

    // Shortcut used to fill the bank with the built-in questions.
    void add(int category, const string& text, const string& a, const string& b,
             const string& c, const string& d, char answer) {
        addQuestion(Question(category, text, a, b, c, d, answer));
    }

public:
    QuestionBank() : count(0), capacity(16) {
        questions = new Question[capacity];
    }

    // Destructor: frees the memory created with new.
    ~QuestionBank() {
        delete[] questions;
    }

    // Copying is disabled so two objects never delete the same array.
    QuestionBank(const QuestionBank&) = delete;
    QuestionBank& operator=(const QuestionBank&) = delete;

    void addQuestion(const Question& question) {
        if (count == capacity) {
            grow();
        }
        questions[count] = question;
        count++;
    }

    int getCount() const { return count; }

    // Counts the questions in one category.
    int countInCategory(int category) const {
        int total = 0;
        for (int i = 0; i < count; i++) {
            if (questions[i].getCategory() == category) {
                total++;
            }
        }
        return total;
    }

    // Fills the bank with the 100 built-in questions (10 per category).
    void loadQuestions() {
        // ---------- 0: Science ----------
        add(0, "What planet is known as the Red Planet?", "Venus", "Mars", "Jupiter", "Mercury", 'B');
        add(0, "What gas do plants absorb from the air?", "Oxygen", "Nitrogen", "Carbon dioxide", "Helium", 'C');
        add(0, "What is the center of an atom called?", "Electron", "Proton", "Nucleus", "Neutron", 'C');
        add(0, "How many bones are in the adult human body?", "186", "206", "226", "256", 'B');
        add(0, "What is the chemical symbol for gold?", "Go", "Gd", "Au", "Ag", 'C');
        add(0, "What is the hardest natural substance?", "Gold", "Iron", "Diamond", "Quartz", 'C');
        add(0, "Which organ produces insulin?", "Liver", "Pancreas", "Kidney", "Heart", 'B');
        add(0, "What is the most common gas in Earth's air?", "Oxygen", "Carbon dioxide", "Nitrogen", "Argon", 'C');
        add(0, "Which planet is closest to the Sun?", "Venus", "Earth", "Mercury", "Mars", 'C');
        add(0, "At what temperature does water boil at sea level?", "90 degrees C", "100 degrees C", "110 degrees C", "120 degrees C", 'B');

        // ---------- 1: History ----------
        add(1, "Who was the first President of the United States?", "Abraham Lincoln", "George Washington", "Thomas Jefferson", "John Adams", 'B');
        add(1, "In which country were the pyramids of Giza built?", "Mexico", "Greece", "Egypt", "Peru", 'C');
        add(1, "In which year did World War II end?", "1918", "1939", "1945", "1950", 'C');
        add(1, "Who was the first person to walk on the Moon?", "Buzz Aldrin", "Yuri Gagarin", "Neil Armstrong", "John Glenn", 'C');
        add(1, "In which year did Columbus first reach the Americas?", "1492", "1500", "1607", "1776", 'A');
        add(1, "In which year was the Declaration of Independence adopted?", "1776", "1789", "1812", "1865", 'A');
        add(1, "Which civilization built Machu Picchu?", "Aztec", "Maya", "Inca", "Olmec", 'C');
        add(1, "In which year did the Berlin Wall fall?", "1985", "1989", "1991", "1993", 'B');
        add(1, "Who was the first emperor of Rome?", "Julius Caesar", "Augustus", "Nero", "Caligula", 'B');
        add(1, "Which ancient city was buried by Mount Vesuvius in 79 AD?", "Athens", "Carthage", "Pompeii", "Sparta", 'C');

        // ---------- 2: Geography ----------
        add(2, "What is the largest ocean on Earth?", "Atlantic", "Indian", "Arctic", "Pacific", 'D');
        add(2, "What is the capital of Australia?", "Sydney", "Melbourne", "Canberra", "Perth", 'C');
        add(2, "How many continents are there on Earth?", "5", "6", "7", "8", 'C');
        add(2, "What is the longest river in South America?", "Amazon", "Parana", "Orinoco", "Sao Francisco", 'A');
        add(2, "What is the smallest country in the world by area?", "Monaco", "Vatican City", "San Marino", "Liechtenstein", 'B');
        add(2, "What is the capital of Canada?", "Toronto", "Vancouver", "Ottawa", "Montreal", 'C');
        add(2, "What is the largest hot desert in the world?", "Gobi", "Sahara", "Kalahari", "Mojave", 'B');
        add(2, "Mount Everest is on the border of Nepal and which country?", "India", "China", "Bhutan", "Pakistan", 'B');
        add(2, "What is the capital of Japan?", "Kyoto", "Osaka", "Tokyo", "Seoul", 'C');
        add(2, "Which U.S. state is the largest by area?", "Texas", "California", "Alaska", "Montana", 'C');

        // ---------- 3: Movies and TV ----------
        add(3, "In Finding Nemo, what kind of fish is Nemo?", "Shark", "Clownfish", "Tuna", "Goldfish", 'B');
        add(3, "Who directed the 1993 movie Jurassic Park?", "James Cameron", "Steven Spielberg", "George Lucas", "Ridley Scott", 'B');
        add(3, "Which 1997 film won the Oscar for Best Picture?", "Titanic", "Good Will Hunting", "As Good as It Gets", "L.A. Confidential", 'A');
        add(3, "In The Lion King (1994), what is Simba's father's name?", "Scar", "Mufasa", "Rafiki", "Timon", 'B');
        add(3, "What is the wizarding school in the Harry Potter films?", "Durmstrang", "Beauxbatons", "Hogwarts", "Ilvermorny", 'C');
        add(3, "Which TV show is set in the town of Hawkins, Indiana?", "Stranger Things", "Riverdale", "Twin Peaks", "Wednesday", 'A');
        add(3, "In The Wizard of Oz (1939), what color are Dorothy's slippers?", "Silver", "Ruby red", "Gold", "Blue", 'B');
        add(3, "Which actor plays Iron Man in the Marvel movies?", "Chris Evans", "Robert Downey Jr.", "Chris Hemsworth", "Mark Ruffalo", 'B');
        add(3, "In the TV show Friends, what is the coffee shop called?", "Central Perk", "Monk's Cafe", "The Grind", "Java Joe's", 'A');
        add(3, "Which animated movie has a snowman named Olaf?", "Tangled", "Moana", "Frozen", "Brave", 'C');

        // ---------- 4: Sports ----------
        add(4, "How many players does one soccer team have on the field?", "9", "10", "11", "12", 'C');
        add(4, "In which sport do you score a touchdown?", "Basketball", "American football", "Baseball", "Hockey", 'B');
        add(4, "How often are the Summer Olympics held?", "Every 2 years", "Every 3 years", "Every 4 years", "Every 5 years", 'C');
        add(4, "In basketball, how many points is a shot from beyond the arc?", "1", "2", "3", "4", 'C');
        add(4, "Which country has won the most FIFA World Cups?", "Germany", "Italy", "Argentina", "Brazil", 'D');
        add(4, "In tennis, what is a score of zero called?", "Nil", "Love", "Zero", "Duck", 'B');
        add(4, "How many holes are in a standard round of golf?", "9", "12", "18", "20", 'C');
        add(4, "In baseball, how many strikes make an out?", "2", "3", "4", "5", 'B');
        add(4, "Which sport uses a shuttlecock?", "Squash", "Badminton", "Table tennis", "Volleyball", 'B');
        add(4, "How long is a marathon?", "26.2 miles", "13.1 miles", "20 miles", "30 miles", 'A');

        // ---------- 5: Basic Math ----------
        add(5, "What is 7 + 8?", "13", "14", "15", "16", 'C');
        add(5, "What is 12 x 12?", "124", "144", "132", "154", 'B');
        add(5, "What is 100 - 37?", "63", "73", "67", "53", 'A');
        add(5, "What is 81 / 9?", "8", "9", "7", "11", 'B');
        add(5, "What is 15 x 4?", "50", "55", "60", "65", 'C');
        add(5, "What is 250 + 175?", "425", "415", "435", "325", 'A');
        add(5, "What is 56 / 7?", "6", "7", "8", "9", 'C');
        add(5, "What is 9 x 7?", "56", "63", "72", "81", 'B');
        add(5, "What is 1000 - 456?", "544", "554", "644", "456", 'A');
        add(5, "What is 6 + 4 x 2? (Remember the order of operations)", "20", "14", "12", "16", 'B');

        // ---------- 6: Literature ----------
        add(6, "Who wrote Romeo and Juliet?", "Charles Dickens", "William Shakespeare", "Jane Austen", "Mark Twain", 'B');
        add(6, "Who wrote Pride and Prejudice?", "Emily Bronte", "Jane Austen", "Mary Shelley", "Virginia Woolf", 'B');
        add(6, "Who wrote the novel 1984?", "Aldous Huxley", "George Orwell", "Ray Bradbury", "H.G. Wells", 'B');
        add(6, "Who wrote The Adventures of Tom Sawyer?", "Mark Twain", "Herman Melville", "Jack London", "Edgar Allan Poe", 'A');
        add(6, "Who wrote Don Quixote?", "Miguel de Cervantes", "Gabriel Garcia Marquez", "Dante Alighieri", "Victor Hugo", 'A');
        add(6, "Who wrote the Harry Potter books?", "J.R.R. Tolkien", "C.S. Lewis", "J.K. Rowling", "Roald Dahl", 'C');
        add(6, "In the novel Moby-Dick, what is Moby Dick?", "A shark", "A whale", "A ship", "A sailor", 'B');
        add(6, "Which ancient poet wrote The Odyssey?", "Homer", "Virgil", "Sophocles", "Plato", 'A');
        add(6, "Who wrote the novel Frankenstein?", "Bram Stoker", "Mary Shelley", "Edgar Allan Poe", "Oscar Wilde", 'B');
        add(6, "Who wrote The Great Gatsby?", "Ernest Hemingway", "F. Scott Fitzgerald", "John Steinbeck", "William Faulkner", 'B');

        // ---------- 7: English ----------
        add(7, "What is the plural of \"child\"?", "Childs", "Childes", "Children", "Childrens", 'C');
        add(7, "Which of these words is a verb?", "Quickly", "Run", "Happy", "Blue", 'B');
        add(7, "What is the opposite of \"ancient\"?", "Old", "Modern", "Historic", "Antique", 'B');
        add(7, "Which word is spelled correctly?", "Recieve", "Receive", "Receeve", "Riceive", 'B');
        add(7, "Which word means the same as \"happy\"?", "Sad", "Joyful", "Angry", "Tired", 'B');
        add(7, "Which of these words is an adjective?", "Beautiful", "Slowly", "Swim", "They", 'A');
        add(7, "What is the past tense of \"go\"?", "Goed", "Gone", "Went", "Going", 'C');
        add(7, "Which sentence is correct?", "Their going home.", "They're going home.", "There going home.", "Theyre going home.", 'B');
        add(7, "Which punctuation mark ends a question?", "Period", "Comma", "Question mark", "Colon", 'C');
        add(7, "What is the opposite of \"generous\"?", "Kind", "Selfish", "Friendly", "Honest", 'B');

        // ---------- 8: Art ----------
        add(8, "Who painted the Mona Lisa?", "Michelangelo", "Leonardo da Vinci", "Raphael", "Vincent van Gogh", 'B');
        add(8, "Who painted The Starry Night?", "Claude Monet", "Vincent van Gogh", "Edvard Munch", "Paul Cezanne", 'B');
        add(8, "What are the three traditional primary colors?", "Red, yellow, blue", "Red, green, blue", "Orange, green, purple", "Black, white, gray", 'A');
        add(8, "Who painted the ceiling of the Sistine Chapel?", "Leonardo da Vinci", "Michelangelo", "Donatello", "Rembrandt", 'B');
        add(8, "Who painted the melting clocks in The Persistence of Memory?", "Salvador Dali", "Pablo Picasso", "Andy Warhol", "Frida Kahlo", 'A');
        add(8, "Pablo Picasso helped start which art style?", "Impressionism", "Cubism", "Surrealism", "Pop Art", 'B');
        add(8, "What color do you get by mixing red and white?", "Pink", "Purple", "Orange", "Brown", 'A');
        add(8, "What color do you get by mixing blue and yellow?", "Green", "Purple", "Orange", "Brown", 'A');
        add(8, "Frida Kahlo was an artist from which country?", "Spain", "Brazil", "Mexico", "Argentina", 'C');
        add(8, "Which artist is famous for paintings of soup cans?", "Andy Warhol", "Roy Lichtenstein", "Keith Haring", "Jackson Pollock", 'A');

        // ---------- 9: Music ----------
        add(9, "How many lines are in a standard music staff?", "4", "5", "6", "7", 'B');
        add(9, "How many keys does a standard piano have?", "76", "88", "96", "100", 'B');
        add(9, "Which famous composer kept writing music after going deaf?", "Mozart", "Bach", "Beethoven", "Chopin", 'C');
        add(9, "Who is known as the King of Pop?", "Elvis Presley", "Michael Jackson", "Prince", "Stevie Wonder", 'B');
        add(9, "Who is known as the King of Rock and Roll?", "Elvis Presley", "Chuck Berry", "Little Richard", "Johnny Cash", 'A');
        add(9, "Which band had John, Paul, George and Ringo?", "The Rolling Stones", "The Beatles", "Queen", "The Who", 'B');
        add(9, "Which instrument family does the trumpet belong to?", "Woodwind", "Brass", "String", "Percussion", 'B');
        add(9, "In music, what does \"forte\" mean?", "Slowly", "Loudly", "Softly", "Quickly", 'B');
        add(9, "Which composer wrote The Four Seasons?", "Vivaldi", "Handel", "Haydn", "Mozart", 'A');
        add(9, "How many musicians are in a quartet?", "2", "3", "4", "5", 'C');
    }

    // Randomly reorders an array of question pointers (Fisher-Yates shuffle).
    // Only the pointers move, so no question is copied or repeated.
    static void shuffle(Question* list[], int size) {
        for (int i = size - 1; i > 0; i--) {
            int j = rand() % (i + 1);
            Question* temp = list[i];
            list[i] = list[j];
            list[j] = temp;
        }
    }

    // Picks "amount" different random questions and stores pointers to them
    // in "selected". Throws runtime_error if the bank doesn't have enough.
    void pickRandomQuestions(Question* selected[], int amount) {
        if (count < amount) {
            throw runtime_error("Not enough questions in the bank.");
        }

        // Temporary dynamic array with a pointer to every question.
        Question** all = new Question*[count];
        for (int i = 0; i < count; i++) {
            all[i] = &questions[i];   // store the address of each question
        }

        shuffle(all, count);

        // Take the first "amount" questions and mix up their options too.
        for (int i = 0; i < amount; i++) {
            selected[i] = all[i];
            selected[i]->shuffleOptions();
        }

        delete[] all;   // free the temporary array
    }
};

// ============================================================
//  Struct + Class: Leaderboard
//  Keeps the top-10 scores, sorted from highest to lowest.
// ============================================================
struct ScoreEntry {
    string name;
    int points;
};

// The scores are kept in memory (not in a file) while the program runs.
// On OnlineGDB, a file would open as an extra tab next to the code.
class Leaderboard {
private:
    ScoreEntry entries[MAX_LEADERS];   // fixed-size array of the top scores
    int count;

    // Insertion sort: highest score first.
    void sortEntries() {
        for (int i = 1; i < count; i++) {
            ScoreEntry current = entries[i];
            int j = i - 1;
            while (j >= 0 && entries[j].points < current.points) {
                entries[j + 1] = entries[j];
                j--;
            }
            entries[j + 1] = current;
        }
    }

public:
    Leaderboard() : count(0) {}

    // Adds a score if it makes the top 10.
    // Returns the rank (1 = first place) or 0 if it didn't make the list.
    int addScore(const string& name, int points) {
        if (count < MAX_LEADERS) {
            entries[count].name = name;
            entries[count].points = points;
            count++;
        } else if (points > entries[count - 1].points) {
            entries[count - 1].name = name;    // replace the lowest score
            entries[count - 1].points = points;
        } else {
            return 0;
        }
        sortEntries();

        // Find where the new score ended up (the last matching entry).
        for (int i = count - 1; i >= 0; i--) {
            if (entries[i].name == name && entries[i].points == points) {
                return i + 1;
            }
        }
        return 0;
    }

    void display() const {
        printTitle("LEADERBOARD - TOP 10");
        if (count == 0) {
            cout << "  No scores yet. Play a round to get on the board!" << endl;
            return;
        }
        cout << "  " << left << setw(6) << "Rank" << setw(MAX_NAME_LENGTH + 2)
             << "Name" << "Points" << endl;
        printLine('-');
        for (int i = 0; i < count; i++) {
            cout << "  " << left << setw(6) << (i + 1)
                 << setw(MAX_NAME_LENGTH + 2) << entries[i].name
                 << entries[i].points << endl;
        }
    }
};

// ============================================================
//  Class: QuizGame
//  Runs the main menu and connects all the other classes.
// ============================================================
class QuizGame {
private:
    QuestionBank bank;
    Leaderboard leaderboard;

    void showMainMenu() const {
        printTitle("QUIZ GAME - MAIN MENU");
        cout << "  1. Play" << endl;
        cout << "  2. View leaderboard" << endl;
        cout << "  3. How to play" << endl;
        cout << "  4. Exit" << endl;
        printLine('-');
    }

    // One full round: 10 random questions, then the results.
    void playRound() {
        printTitle("NEW GAME");
        string name = askName("Enter your name: ");

        // Array of pointers to the 10 questions chosen for this round.
        Question* selected[QUESTIONS_PER_ROUND];
        bank.pickRandomQuestions(selected, QUESTIONS_PER_ROUND);

        // Feedback on the last answer, shown at the top of the next screen.
        string lastFeedback = "";

        Player player(name);
        cout << endl << "Good luck, " << name << "! Here come "
             << QUESTIONS_PER_ROUND << " random questions." << endl;

        for (int i = 0; i < QUESTIONS_PER_ROUND; i++) {
            Question* current = selected[i];   // pointer to this question
            clearScreen();
            player.displayHUD(i + 1, QUESTIONS_PER_ROUND, current->getCategoryName());
            if (!lastFeedback.empty()) {
                cout << lastFeedback;
            }
            current->display();

            char answer = askAnswer("\n  Your answer (A-D): ");

            if (current->isCorrect(answer)) {
                int bonus = player.recordCorrect(current->getCategory());
                lastFeedback = "  Last answer: " + green("Correct! +" +
                               to_string(POINTS_PER_QUESTION) + " points.") + "\n";
                if (bonus > 0) {
                    lastFeedback += "  " + yellow(to_string(player.getStreak()) +
                                    " in a row! Streak bonus +" +
                                    to_string(bonus) + " points.") + "\n";
                }
            } else {
                player.recordWrong(current->getCategory());
                lastFeedback = "  Last answer: " + red(string("Wrong. The correct answer was ") +
                               current->getCorrectLetter() + ") " +
                               current->getCorrectOptionText() + ".") + "\n";
            }
            // No pause here: the next question appears right away,
            // with the feedback for this answer at the top.
        }
        clearScreen();
        cout << lastFeedback;   // feedback for the last question

        showResults(player);
        saveScore(player);
    }

    // Shows the final score, grade and category breakdown.
    void showResults(const Player& player) const {
        printTitle("RESULTS");
        cout << "  Player:        " << player.getName() << endl;
        cout << "  Correct:       " << player.getCorrectAnswers() << " of "
             << player.getQuestionsAnswered() << endl;
        cout << "  Percentage:    " << fixed << setprecision(0)
             << player.getPercentage() << "%" << endl;
        cout << "  Grade:         " << player.getGrade() << endl;
        cout << "  Points:        " << player.getPoints() << endl;
        cout << "  Best streak:   " << player.getBestStreak() << endl;

        player.displayCategoryBreakdown();

        cout << endl;
        switch (player.getGrade()) {
            case 'A': cout << "  Outstanding! You really know your stuff." << endl; break;
            case 'B': cout << "  Great job!" << endl; break;
            case 'C': cout << "  Good effort. Keep practicing!" << endl; break;
            case 'D': cout << "  Not bad, but there's room to improve." << endl; break;
            default:  cout << "  Tough round. Try again!" << endl; break;
        }
    }

    // Adds the score to the leaderboard.
    void saveScore(const Player& player) {
        int rank = leaderboard.addScore(player.getName(), player.getPoints());
        if (rank > 0) {
            cout << endl << "  New high score! You are #" << rank
                 << " on the leaderboard." << endl;
        } else {
            cout << endl << "  Not enough for the top " << MAX_LEADERS
                 << " this time. Try again!" << endl;
        }
    }

    void showLeaderboard() const {
        leaderboard.display();
        pressEnterToContinue();
    }

    void showRules() const {
        printTitle("HOW TO PLAY");
        cout << "  1. Choose Play and enter your name." << endl;
        cout << "  2. You get " << QUESTIONS_PER_ROUND << " random questions from a pool of "
             << bank.getCount() << "." << endl;
        cout << "  3. Answer each question by typing A, B, C or D." << endl;
        cout << "  4. Each correct answer is worth " << POINTS_PER_QUESTION
             << " points." << endl;
        cout << "  5. Every " << STREAK_FOR_BONUS << " correct answers in a row earns a "
             << STREAK_BONUS << "-point bonus." << endl;
        cout << "  6. After each answer you see if you were right, and the" << endl;
        cout << "     correct answer if you were wrong." << endl;
        cout << "  7. At the end you see your grade and your results by category." << endl;
        cout << "  8. The top " << MAX_LEADERS << " scores stay on the leaderboard" << endl;
        cout << "     while the game is running." << endl;
        cout << endl << "  Categories:" << endl;
        for (int i = 0; i < NUM_CATEGORIES; i++) {
            cout << "    - " << CATEGORIES[i] << endl;
        }
        pressEnterToContinue();
    }

public:
    // Loads the 100 questions.
    void start() {
        printTitle("WELCOME TO THE QUIZ GAME");
        bank.loadQuestions();
        cout << "Loaded " << bank.getCount() << " questions in "
             << NUM_CATEGORIES << " categories." << endl;
    }

    // Main menu loop. Runs until the player chooses Exit.
    void run() {
        bool running = true;
        while (running) {
            showMainMenu();
            int choice = askInt("Choose an option (1-4): ", 1, 4);

            try {
                switch (choice) {
                    case 1: playRound();       break;
                    case 2: showLeaderboard(); break;
                    case 3: showRules();       break;
                    case 4: running = false;   break;
                }
            } catch (const exception& e) {
                // Any unexpected problem is reported instead of crashing.
                cout << endl << "  Error: " << e.what() << endl;
                pressEnterToContinue();
            }
        }
        cout << endl << "Thanks for playing. Goodbye!" << endl;
    }
};

// ============================================================
//  main
// ============================================================
int main() {
    srand(static_cast<unsigned int>(time(0)));   // new random order every run

    QuizGame game;
    game.start();
    game.run();
    return 0;
    }
