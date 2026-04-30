#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <ctype.h>
#include <time.h>

// đọc IPA
#ifdef _WIN32
#include <windows.h>
#endif

#define MAX_WORD 55
#define MAX_TYPE 20
#define MAX_PRON 50
#define MAX_MEAN 200
#define HASH_SIZE 100
#define FILENAME "dictionary.txt"
#define PROGRESS_FILE "progress.txt"

// Bảng xếp hạng
#define MAX_SCORES 5
#define MAX_NAME 30

typedef struct Word {
    char tuvung[MAX_WORD];
    char loaitu[MAX_TYPE];
    char phienam[MAX_PRON];
    char nghia[MAX_MEAN];
    
    // Thuộc tính cho Flashcard
    time_t nextReview; 
    int interval;
    int easeFactor;
} Word;
//node
typedef struct Node {
    Word data;
    struct Node* next;
} Node;

//BXH
typedef struct {
    char name[MAX_NAME];
    int score;
} HighScore;

Node* hashTable[HASH_SIZE] = {NULL};
HighScore highScores[MAX_SCORES];
int highScoreCount = 0;
char playerName[MAX_NAME] = "";

//HÀM TIỆN ÍCH
void toLowerCase(char* str) {
    for (int i = 0; str[i]; i++) {
        str[i] = tolower(str[i]);
    }
}

void clearBuffer() {
    int c;
    while ((c = getchar()) != '\n' && c != EOF);
}

//HÀM BĂM & QUẢN LÝ TỪ ĐIỂN
int hash(char* str) {
    unsigned long sum = 0;
    for (int i = 0; str[i]; i++) {
        sum += toupper(str[i]);
    }
    return sum % HASH_SIZE;
}

Node* searchWord(char* wordInput) {
    char lowerWord[MAX_WORD];
    strcpy(lowerWord, wordInput);
    toLowerCase(lowerWord);
    
    int index = hash(lowerWord);
    Node* current = hashTable[index];
    while (current != NULL) {
        if (strcmp(current->data.tuvung, lowerWord) == 0)
            return current;
        current = current->next;
    }
    return NULL;
}
// Thêm từ
void insertWord(Word w, int silent) {
    char lowerWord[MAX_WORD];
    strcpy(lowerWord, w.tuvung);
    toLowerCase(lowerWord);
    
    if (searchWord(lowerWord) != NULL) {
        if (!silent) printf("Tu da ton tai!\n");
        return;
    }
    
    int index = hash(lowerWord);
    Node* newNode = (Node*)malloc(sizeof(Node));
    
    strcpy(newNode->data.tuvung, lowerWord);
    strcpy(newNode->data.loaitu, w.loaitu);
    strcpy(newNode->data.phienam, w.phienam);
    strcpy(newNode->data.nghia, w.nghia);
    
    // Khởi tạo Flashcard
    newNode->data.nextReview = 0;
    newNode->data.interval = 0;
    newNode->data.easeFactor = 250; 
    
    newNode->next = hashTable[index];
    hashTable[index] = newNode;
    if (!silent) printf("Da them tu thanh cong.\n");
}

//Xóa từ
int deleteWord(char* wordInput) {
    char lowerWord[MAX_WORD];
    strcpy(lowerWord, wordInput);
    toLowerCase(lowerWord);
    
    int index = hash(lowerWord);
    Node* current = hashTable[index];
    Node* prev = NULL;
    while (current != NULL) {
        if (strcmp(current->data.tuvung, lowerWord) == 0) {
            if (prev == NULL) hashTable[index] = current->next;
            else prev->next = current->next;
            free(current);
            return 1;
        }
        prev = current;
        current = current->next;
    }
    return 0;
}

//LƯU & TẢI DỮ LIỆU
void loadDictionary(char* filename) {
    FILE* f = fopen(filename, "r");
    if (f == NULL) return;

    char line[400];
    while (fgets(line, sizeof(line), f)) {
        line[strcspn(line, "\n")] = '\0';
        char* token;
        Word w;
        
        token = strtok(line, "|"); if (token == NULL) continue;
        strcpy(w.tuvung, token); toLowerCase(w.tuvung);
        
        token = strtok(NULL, "|"); if (token == NULL) continue;
        strcpy(w.loaitu, token);
        
        token = strtok(NULL, "|"); if (token == NULL) continue;
        strcpy(w.phienam, token);
        
        token = strtok(NULL, "|"); if (token == NULL) continue;
        strcpy(w.nghia, token);
        
        insertWord(w, 1);
    }
    fclose(f);
}

void saveDictionary(char* filename) {
    FILE* f = fopen(filename, "w");
    if (f == NULL) {
        printf("Loi mo file tu dien de ghi!\n");
        return;
    }
    for (int i = 0; i < HASH_SIZE; i++) {
        Node* current = hashTable[i];
        while (current != NULL) {
            fprintf(f, "%s|%s|%s|%s\n",
                    current->data.tuvung, current->data.loaitu,
                    current->data.phienam, current->data.nghia);
            current = current->next;
        }
    }
    fclose(f);
}

void loadProgress() {
    FILE* f = fopen(PROGRESS_FILE, "r");
    if (f == NULL) return;

    char line[200];
    while (fgets(line, sizeof(line), f)) {
        line[strcspn(line, "\n")] = '\0';
        char wordInput[MAX_WORD];
        long nextRev; 
        int interval, ease;
        
        char* token = strtok(line, "|"); if (!token) continue;
        strcpy(wordInput, token);
        
        token = strtok(NULL, "|"); if (!token) continue; nextRev = atol(token);
        token = strtok(NULL, "|"); if (!token) continue; interval = atoi(token);
        token = strtok(NULL, "|"); if (!token) continue; ease = atoi(token);

        Node* node = searchWord(wordInput);
        if (node) {
            node->data.nextReview = (time_t)nextRev;
            node->data.interval = interval;
            node->data.easeFactor = ease;
        }
    }
    fclose(f);
}

void saveProgress() {
    FILE* f = fopen(PROGRESS_FILE, "w");
    if (f == NULL) return;
    
    for (int i = 0; i < HASH_SIZE; i++) {
        Node* current = hashTable[i];
        while (current != NULL) {
            fprintf(f, "%s|%ld|%d|%d\n", 
                    current->data.tuvung, (long)current->data.nextReview, 
                    current->data.interval, current->data.easeFactor);
            current = current->next;
        }
    }
    fclose(f);
}

void loadHighScores() {
    FILE* f = fopen("scores.txt", "r");
    if (!f) return;
    highScoreCount = 0;
    while (highScoreCount < MAX_SCORES &&
           fscanf(f, "%[^|]|%d\n", highScores[highScoreCount].name,
                  &highScores[highScoreCount].score) == 2) {
        highScoreCount++;
    }
    fclose(f);
}

void saveHighScores() {
    FILE* f = fopen("scores.txt", "w");
    if (!f) return;
    for (int i = 0; i < highScoreCount; i++) {
        fprintf(f, "%s|%d\n", highScores[i].name, highScores[i].score);
    }
    fclose(f);
}

void freeHashTable() {
    for (int i = 0; i < HASH_SIZE; i++) {
        Node* current = hashTable[i];
        while (current) {
            Node* temp = current;
            current = current->next;
            free(temp);
        }
        hashTable[i] = NULL;
    }
}

//CÁC CHỨC NĂNG CHÍNH
void addWord() {
    Word w;
    printf("\n--- THEM TU MOI ---\n");
    printf("Nhap tu: ");
    fgets(w.tuvung, MAX_WORD, stdin);
    w.tuvung[strcspn(w.tuvung, "\n")] = '\0';
    toLowerCase(w.tuvung);

    printf("Nhap loai tu (n, v, adj,...): ");
    fgets(w.loaitu, MAX_TYPE, stdin);
    w.loaitu[strcspn(w.loaitu, "\n")] = '\0';

    printf("Nhap phien am: ");
    fgets(w.phienam, MAX_PRON, stdin);
    w.phienam[strcspn(w.phienam, "\n")] = '\0';

    printf("Nhap nghia (cac nghia cach nhau boi dau phay): ");
    fgets(w.nghia, MAX_MEAN, stdin);
    w.nghia[strcspn(w.nghia, "\n")] = '\0';

    insertWord(w, 0);
    saveDictionary(FILENAME);
}

void viewWordDetail() {
    char wordInput[MAX_WORD];
    clearBuffer();
    printf("Nhap tu can xem: ");
    fgets(wordInput, MAX_WORD, stdin);
    wordInput[strcspn(wordInput, "\n")] = '\0';
    toLowerCase(wordInput);

    Node* node = searchWord(wordInput);
    if (node == NULL) {
        printf("Khong tim thay tu.\n");
        return;
    }
    printf("\nTu: %s\n", node->data.tuvung);
    printf("Loai tu: %s\n", node->data.loaitu);
    printf("Phien am: %s\n", node->data.phienam);
    printf("Nghia: %s\n", node->data.nghia);
}

void editWord() {
    char wordInput[MAX_WORD];
    clearBuffer();
    printf("Nhap tu can sua: ");
    fgets(wordInput, MAX_WORD, stdin);
    wordInput[strcspn(wordInput, "\n")] = '\0';
    toLowerCase(wordInput);

    Node* node = searchWord(wordInput);
    if (node == NULL) {
        printf("Khong tim thay tu.\n");
        return;
    }

    printf("Nhap thong tin moi (de trong neu giu nguyen):\n");
    Word newWord = node->data;

    char buffer[MAX_MEAN];
    printf("Loai tu [%s]: ", newWord.loaitu);
    fgets(buffer, MAX_TYPE, stdin);
    buffer[strcspn(buffer, "\n")] = '\0';
    if (strlen(buffer) > 0) strcpy(newWord.loaitu, buffer);

    printf("Phien am [%s]: ", newWord.phienam);
    fgets(buffer, MAX_PRON, stdin);
    buffer[strcspn(buffer, "\n")] = '\0';
    if (strlen(buffer) > 0) strcpy(newWord.phienam, buffer);

    printf("Nghia [%s]: ", newWord.nghia);
    fgets(buffer, MAX_MEAN, stdin);
    buffer[strcspn(buffer, "\n")] = '\0';
    if (strlen(buffer) > 0) strcpy(newWord.nghia, buffer);

    node->data = newWord;
    saveDictionary(FILENAME);
    printf("Da cap nhat.\n");
}

void removeWord() {
    char wordInput[MAX_WORD];
    clearBuffer();
    printf("Nhap tu can xoa: ");
    fgets(wordInput, MAX_WORD, stdin);
    wordInput[strcspn(wordInput, "\n")] = '\0';
    toLowerCase(wordInput);

    if (deleteWord(wordInput)) {
        saveDictionary(FILENAME);
        printf("Da xoa tu.\n");
    } else {
        printf("Khong tim thay tu.\n");
    }
}

void showDictionary() {
    int choice;
    do {
        printf("\n========== DANH SACH TU DIEN ==========\n");
        int count = 0, col = 0;
        for (int i = 0; i < HASH_SIZE; i++) {
            Node* current = hashTable[i];
            while (current != NULL) {
                printf("%-15s", current->data.tuvung);
                col++;
                if (col % 5 == 0) printf("\n");
                count++;
                current = current->next;
            }
        }
        if (count == 0) printf("(Tu dien trong)");
        printf("\nTong so: %d tu\n", count);

        printf("\n--- THAO TAC TU DIEN ---\n");
        printf("1. Xem chi tiet mot tu\n");
        printf("2. Sua thong tin mot tu\n");
        printf("3. Xoa mot tu\n");
        printf("4. Quay lai\n");
        printf("Chon: ");
        scanf("%d", &choice);

        switch (choice) {
            case 1: viewWordDetail(); break;
            case 2: editWord(); break;
            case 3: removeWord(); break;
            case 4: printf("Quay lai menu truoc.\n"); break;
            default: printf("Lua chon khong hop le.\n");
        }
    } while (choice != 4);
}

void listWordsByType() {
    char type[MAX_TYPE];
    printf("\nNhap loai tu can thong ke (n, v, adj,...): ");
    fgets(type, MAX_TYPE, stdin);
    type[strcspn(type, "\n")] = '\0';

    printf("\nCac tu co loai '%s':\n", type);
    int found = 0;
    for (int i = 0; i < HASH_SIZE; i++) {
        Node* cur = hashTable[i];
        while (cur) {
            if (strcmp(cur->data.loaitu, type) == 0) {
                printf("%s - %s : %s\n", cur->data.tuvung, cur->data.phienam, cur->data.nghia);
                found++;
            }
            cur = cur->next;
        }
    }
    if (!found) printf("Khong co tu nao thuoc loai nay.\n");
}

void addHighScore(char* name, int score) {
    int pos = highScoreCount;
    for (int i = 0; i < highScoreCount; i++) {
        if (score > highScores[i].score) {
            pos = i;
            break;
        }
    }
    if (pos < MAX_SCORES) {
        for (int i = highScoreCount; i > pos && i < MAX_SCORES; i--) {
            highScores[i] = highScores[i-1];
        }
        strcpy(highScores[pos].name, name);
        highScores[pos].score = score;
        if (highScoreCount < MAX_SCORES) highScoreCount++;
    }
    saveHighScores();
}

void showHighScores() {
    printf("\n===== BANG XEP HANG =====\n");
    if (highScoreCount == 0) {
        printf("Chua co nguoi choi.\n");
    } else {
        for (int i = 0; i < highScoreCount; i++) {
            printf("%d. %s - %d diem\n", i+1, highScores[i].name, highScores[i].score);
        }
    }
}

//TRÒ CHƠI
void playGameFillChar() {
    if (strlen(playerName) == 0) {
        printf("\nNhap ten cua ban: ");
        fgets(playerName, MAX_NAME, stdin);
        playerName[strcspn(playerName, "\n")] = '\0';
    }

    int totalWords = 0;
    for (int i = 0; i < HASH_SIZE; i++) {
        Node* cur = hashTable[i];
        while (cur) { totalWords++; cur = cur->next; }
    }
    if (totalWords == 0) {
        printf("Tu dien trong, khong the choi.\n");
        return;
    }

    char choice;
    int currentScore = 0;
    do {
        int randomIndex = rand() % totalWords;
        Node* selected = NULL;
        int count = 0;
        for (int i = 0; i < HASH_SIZE; i++) {
            Node* cur = hashTable[i];
            while (cur) {
                if (count == randomIndex) { selected = cur; break; }
                count++; cur = cur->next;
            }
            if (selected) break;
        }

        char original[MAX_WORD];
        strcpy(original, selected->data.tuvung);
        int len = strlen(original);
        if (len == 0) continue;
        
        int pos;
        do { pos = rand() % len; } while (original[pos] == ' '); 

        char question[MAX_WORD];
        strcpy(question, original);
        question[pos] = '_';

        printf("\nTu can doan: %s\n", question);
        printf("Nhap ky tu con thieu: ");
        char ch;
        scanf(" %c", &ch);
        ch = tolower(ch);

        char newWord[MAX_WORD];
        strcpy(newWord, original);
        newWord[pos] = ch;

        Node* found = searchWord(newWord);
        
        // Bỏ điều kiện && strcmp(newWord, original) == 0
        if (found != NULL) {
            printf("Chuc mung! Tu dung la: %s\n", newWord);
            printf("Nghia: %s\n", found->data.nghia);
            currentScore++;
            printf("Diem hien tai: %d\n", currentScore);
        } else {
            printf("Khong co tu nay trong tieng Anh\n");
        }

        printf("Choi tiep? (c/k): ");
        scanf(" %c", &choice);
        clearBuffer(); // Xóa đệm sau khi nhập c/k
    } while (choice == 'c' || choice == 'C');

    if (currentScore > 0) {
        addHighScore(playerName, currentScore);
        printf("Da luu diem %d cua ban vao bang xep hang.\n", currentScore);
    }
}


void playGameByMeaning() {
    if (strlen(playerName) == 0) {
        printf("\nNhap ten cua ban: ");
        fgets(playerName, MAX_NAME, stdin);
        playerName[strcspn(playerName, "\n")] = '\0';
    }

    int totalWords = 0;
    for (int i = 0; i < HASH_SIZE; i++) {
        Node* cur = hashTable[i];
        while (cur) { totalWords++; cur = cur->next; }
    }
    if (totalWords == 0) {
        printf("Tu dien trong, khong the choi.\n");
        return;
    }

    char choice;
    int currentScore = 0;
    do {
        int randomIndex = rand() % totalWords;
        Node* selected = NULL;
        int count = 0;
        for (int i = 0; i < HASH_SIZE; i++) {
            Node* cur = hashTable[i];
            while (cur) {
                if (count == randomIndex) { selected = cur; break; }
                count++; cur = cur->next;
            }
            if (selected) break;
        }

        printf("\nNghia cua tu can doan: %s\n", selected->data.nghia);
        printf("Nhap tu (khong phan biet hoa thuong): ");
        char guess[MAX_WORD];
        
        fgets(guess, MAX_WORD, stdin);
        guess[strcspn(guess, "\n")] = '\0';
        toLowerCase(guess);

        if (strcmp(guess, selected->data.tuvung) == 0) {
            printf("Chinh xac! Tu dung la: %s\n", selected->data.tuvung);
            currentScore++;
            printf("Diem hien tai: %d\n", currentScore);
        } else {
            printf("Sai roi. Dap an dung la: %s\n", selected->data.tuvung);
        }

        printf("Choi tiep? (c/k): ");
        scanf(" %c", &choice);
        clearBuffer(); // Xóa đệm sau khi nhập c/k
    } while (choice == 'c' || choice == 'C');

    if (currentScore > 0) {
        addHighScore(playerName, currentScore);
        printf("Da luu diem %d cua ban vao bang xep hang.\n", currentScore);
    }
}

//FLASHCARD (HỌC LẶP LẠI NGẮT QUÃNG) 
void playFlashcard() {
    time_t now = time(NULL);
    
    // Đếm số từ cần ôn tập ban đầu
    int initialDueCount = 0;
    for (int i = 0; i < HASH_SIZE; i++) {
        Node* cur = hashTable[i];
        while (cur) {
            if (cur->data.nextReview <= now) initialDueCount++;
            cur = cur->next;
        }
    }
    
    if (initialDueCount == 0) {
        printf("\nKhong co tu nao can on tap vao luc nay. Ban da hoc rat cham chi!\n");
        return;
    }
    
    printf("\n===== FLASHCARD - ON TAP THONG MINH =====\n");
    printf("Co %d tu dang cho ban on tap hom nay.\n", initialDueCount);
    printf("Sau khi nhap tu, hay danh gia muc do nho (0-3):\n");
    printf("  0 - Quen hoan toan (Se lap lai & dua vao muc On Tap)\n");
    printf("  1 - Kho nho       (Se lap lai & dua vao muc On Tap)\n");
    printf("  2 - Nho hoi lau   (Xong cho hom nay, chuyen sang ngay mai)\n");
    printf("  3 - Nho ngay      (Xong cho hom nay, chuyen sang vai ngay sau)\n\n");
    printf("Nhan Enter de bat dau...");
    clearBuffer();
    
    // Mảng để lưu các từ user chọn 0-1
    char wordsToReview[HASH_SIZE][MAX_WORD];
    int reviewCount = 0;
    
    char choice;
    do {
        // MỖI VÒNG LẶP: Quét lại danh sách để loại bỏ các từ đã đánh giá 2-3
        Node* selected = NULL;
        int currentDueCount = 0;
        int cnt = 0;
        
        for (int i = 0; i < HASH_SIZE; i++) {
            Node* cur = hashTable[i];
            while (cur) {
                if (cur->data.nextReview <= now) {
                    currentDueCount++;
                    // Thuật toán chọn ngẫu nhiên 1 từ trong số các từ đang tới hạn
                    if (cnt == 0 || rand() % (cnt + 1) == 0) selected = cur;
                    cnt++;
                }
                cur = cur->next;
            }
        }
        
        // Nếu không còn từ nào <= now, tức là đã đánh giá 2-3 hết
        if (!selected) {
            printf("\nHoan ho! Ban da hoan thanh tat ca cac tu can on tap hom nay!\n");
            break;
        }
        
        printf("\n[%d tu con lai] NGHIA: %s\n", currentDueCount, selected->data.nghia);
        printf("Nhap tu: ");
        char guess[MAX_WORD];
        fgets(guess, MAX_WORD, stdin);
        guess[strcspn(guess, "\n")] = '\0';
        toLowerCase(guess);
        
        if (strcmp(guess, selected->data.tuvung) == 0) {
            printf("Chinh xac!\n");
        } else {
            printf("Sai! Dap an dung: %s\n", selected->data.tuvung);
            printf("Thong tin: %s (%s) - %s\n", 
                   selected->data.tuvung, selected->data.phienam, selected->data.nghia);
        }
        
        int rating;
        printf("Danh gia muc do nho (0-3): ");
        scanf("%d", &rating);
        
        if (rating >= 2) { // User chọn 2 hoặc 3
            if (rating == 3) {
                selected->data.easeFactor += 15;
                if (selected->data.easeFactor > 300) selected->data.easeFactor = 300;
                selected->data.interval = (selected->data.interval == 0) ? 1 : (int)(selected->data.interval * (selected->data.easeFactor / 100.0));
            } else { // rating == 2
                selected->data.interval = (selected->data.interval == 0) ? 1 : selected->data.interval;
            }
            
            // Đẩy nextReview tới tương lai -> Từ này sẽ KHÔNG hiện ra nữa trong hôm nay
            selected->data.nextReview = now + selected->data.interval * 24 * 3600;
            printf("=> Tot lam! Tu nay da hoan thanh trong hom nay.\n");
            
        } else { // User chọn 0 hoặc 1
            selected->data.interval = 0;
            selected->data.easeFactor -= 20;
            if (selected->data.easeFactor < 130) selected->data.easeFactor = 130;
            
            // Giữ nguyên nextReview = now -> Từ này SẼ TIẾP TỤC BỊ LẶP LẠI
            selected->data.nextReview = now; 
            
            // Lưu vào danh sách ôn tập (kiểm tra trùng lặp trước khi thêm)
            int alreadyInList = 0;
            for (int k = 0; k < reviewCount; k++) {
                if (strcmp(wordsToReview[k], selected->data.tuvung) == 0) {
                    alreadyInList = 1; 
                    break;
                }
            }
            if (!alreadyInList) {
                strcpy(wordsToReview[reviewCount], selected->data.tuvung);
                reviewCount++;
            }
            printf("=> Da luu '%s' vao danh sach can on tap (se lap lai).\n", selected->data.tuvung);
        }
        
        printf("\nBan muon tiep tuc on tap? (c/k): ");
        scanf(" %c", &choice);
        clearBuffer();
    } while (choice == 'c' || choice == 'C');
    
    // TỔNG KẾT MỤC ÔN TẬP
    if (reviewCount > 0) {
        printf("\n====================================\n");
        printf("   DANH SACH TU YEU CAN ON TAP THEM   \n");
        printf("====================================\n");
        for (int i = 0; i < reviewCount; i++) {
            Node* info = searchWord(wordsToReview[i]);
            if(info) {
                printf("- %s (%s) : %s\n", info->data.tuvung, info->data.phienam, info->data.nghia);
            }
        }
        printf("====================================\n");
    }
    
    saveProgress();
    printf("Tien do flashcard da duoc luu.\n");
}

//CÁC MENU PHỤ
void subMenuDictionary() {
    int choice;
    do {
        printf("\n--- 2. XEM TU DIEN ---\n");
        printf("1. Hien thi chi tiet va cap nhat tu dien\n");
        printf("2. Thong ke tu theo loai (n, v, adj...)\n");
        printf("3. Quay lai menu chinh\n");
        printf("Chon: ");
        scanf("%d", &choice);
        clearBuffer(); 

        switch (choice) {
            case 1: showDictionary(); break;
            case 2: listWordsByType(); break;
            case 3: printf("Dang quay lai...\n"); break;
            default: printf("Lua chon khong hop le.\n");
        }
    } while (choice != 3);
}

void subMenuGames() {
    int choice;
    do {
        printf("\n--- 3. CAC TRO CHOI ---\n");
        printf("1. Tro choi dien ky tu\n");
        printf("2. Tro choi doan nghia\n");
        printf("3. Quay lai menu chinh\n");
        printf("Chon: ");
        scanf("%d", &choice);
        clearBuffer(); 

        switch (choice) {
            case 1: playGameFillChar(); break;
            case 2: playGameByMeaning(); break;
            case 3: printf("Dang quay lai...\n"); break;
            default: printf("Lua chon khong hop le.\n");
        }
    } while (choice != 3);
}

int main() {

    int leTrai = 12; 
    int leMenu = leTrai + 14;

    printf("\n");
    // In khung Header
    printf("%*s|-----------------------------------------------------------------|\n", leTrai, "");
    printf("%*s|           Truong: Dai Hoc Bach Khoa - Dai hoc Da Nang           |\n", leTrai, "");
    printf("%*s|                    Khoa: Cong nghe thong tin                    |\n", leTrai, "");
    printf("%*s|                 PBL1: DO AN LAP TRINH TINH TOAN                 |\n", leTrai, "");
    printf("%*s|        De tai: Hoc tu vung tieng Anh thong qua tro choi         |\n", leTrai, "");
    printf("%*s|              Giao vien huong dan: Truong Ngoc Chau              |\n", leTrai, "");
    printf("%*s|              Sinh vien: Nguyen Hung Thinh - 25T_Nhat1           |\n", leTrai, "");
    printf("%*s|                         Huynh Thi Anh Ngoc - 25T_Nhat1          |\n", leTrai, "");
    printf("%*s|-----------------------------------------------------------------|\n", leTrai, "");

#ifdef _WIN32
    SetConsoleOutputCP(CP_UTF8);
    SetConsoleCP(CP_UTF8);
#endif

    srand((unsigned int)time(NULL));
    loadDictionary(FILENAME);
    loadProgress();
    loadHighScores();

    int choice;
    do {
    printf("%*s|--------------------------------------------|\n", leMenu, "");
    printf("%*s|          CHUONG TRINH HOC TU VUNG          |\n", leMenu, "");
    printf("%*s|--------------------------------------------|\n", leMenu, "");
    printf("%*s|           1. Them tu moi                   |\n", leMenu, "");
    printf("%*s|           2. Xem tu dien                   |\n", leMenu, "");
    printf("%*s|           3. Cac tro choi                  |\n", leMenu, "");
    printf("%*s|           4. Hoc Flashcard                 |\n", leMenu, "");
    printf("%*s|           5. Bang xep hang                 |\n", leMenu, "");
    printf("%*s|           6. Luu va Thoat                  |\n", leMenu, "");
    printf("%*s|--------------------------------------------|\n", leMenu, "");
    
    // Đẩy chữ "Chon chuc nang" thụt vào trong một chút cho thẳng hàng với cột số 1,2,3
    printf("\n%*sChon chuc nang (1-6): ", leMenu + 8, "");
        scanf("%d", &choice);
        clearBuffer(); 

        switch (choice) {
            case 1: addWord(); break;
            case 2: subMenuDictionary(); break;
            case 3: subMenuGames(); break;
            case 4: playFlashcard(); break;
            case 5: showHighScores(); break;
            case 6: 
                saveDictionary(FILENAME);
                saveProgress();
                freeHashTable();
                printf("\nDa luu toan bo du lieu. Tam biet!\n");
                break;
            default: 
                printf("Lua chon khong hop le. Vui long nhap tu 1 den 6.\n");
        }
    } while (choice != 6);

    return 0;
}
