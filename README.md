# networkpractice

# PCCSL507 Networks Lab --- Experiments 5--9

**Only the program code is included.** Each experiment has separate
server and client programs. Use the code-block copy control to copy
either program.

> Compile and run the server in one terminal, then compile and run the
> client in another. These programs target Linux/POSIX systems.

------------------------------------------------------------------------

## Experiment 5 --- TCP Matrix Classification

### `server.c`

``` c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

#define PORT 8080
#define MAX 20

int receive_all(int fd, void *data, size_t size)
{
    size_t total = 0;
    while (total < size) {
        ssize_t n = recv(fd, (char *)data + total, size - total, 0);
        if (n <= 0) return 0;
        total += n;
    }
    return 1;
}

int send_all(int fd, const void *data, size_t size)
{
    size_t total = 0;
    while (total < size) {
        ssize_t n = send(fd, (const char *)data + total, size - total, 0);
        if (n <= 0) return 0;
        total += n;
    }
    return 1;
}

int main(void)
{
    int server_fd, client_fd, n;
    int matrix[MAX][MAX] = {0};
    char result[50];
    struct sockaddr_in addr = {0};
    socklen_t addr_len = sizeof(addr);

    server_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (server_fd < 0) { perror("socket"); return 1; }

    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = INADDR_ANY;
    addr.sin_port = htons(PORT);

    if (bind(server_fd, (struct sockaddr *)&addr, sizeof(addr)) < 0 ||
        listen(server_fd, 3) < 0) {
        perror("bind/listen");
        close(server_fd);
        return 1;
    }

    printf("Waiting for client connection...\n");
    client_fd = accept(server_fd, (struct sockaddr *)&addr, &addr_len);
    if (client_fd < 0) { perror("accept"); close(server_fd); return 1; }

    if (!receive_all(client_fd, &n, sizeof(n)) || n < 1 || n > MAX ||
        !receive_all(client_fd, matrix, sizeof(matrix))) {
        fprintf(stderr, "Invalid or incomplete matrix received.\n");
        close(client_fd);
        close(server_fd);
        return 1;
    }

    int upper = 1, lower = 1, diagonal = 1;

    for (int i = 0; i < n; i++) {
        for (int j = 0; j < n; j++) {
            if (i > j && matrix[i][j] != 0) upper = 0;
            if (i < j && matrix[i][j] != 0) lower = 0;
            if (i != j && matrix[i][j] != 0) diagonal = 0;
        }
    }

    if (diagonal)
        strcpy(result, "Diagonal Matrix");
    else if (upper)
        strcpy(result, "Upper Triangular Matrix");
    else if (lower)
        strcpy(result, "Lower Triangular Matrix");
    else
        strcpy(result, "Neither Upper nor Lower nor Diagonal");

    send_all(client_fd, result, sizeof(result));
    printf("Matrix Type Sent to Client: %s\n", result);

    close(client_fd);
    close(server_fd);
    return 0;
}
```

### `client.c`

``` c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <unistd.h>
#include <arpa/inet.h>

#define PORT 8080
#define MAX 20

int send_all(int fd, const void *data, size_t size)
{
    size_t total = 0;
    while (total < size) {
        ssize_t n = send(fd, (const char *)data + total, size - total, 0);
        if (n <= 0) return 0;
        total += n;
    }
    return 1;
}

int receive_all(int fd, void *data, size_t size)
{
    size_t total = 0;
    while (total < size) {
        ssize_t n = recv(fd, (char *)data + total, size - total, 0);
        if (n <= 0) return 0;
        total += n;
    }
    return 1;
}

int main(void)
{
    int sock, n, matrix[MAX][MAX] = {0};
    char result[50] = {0};
    struct sockaddr_in server = {0};

    sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock < 0) { perror("socket"); return 1; }

    server.sin_family = AF_INET;
    server.sin_port = htons(PORT);
    inet_pton(AF_INET, "127.0.0.1", &server.sin_addr);

    if (connect(sock, (struct sockaddr *)&server, sizeof(server)) < 0) {
        perror("connect");
        close(sock);
        return 1;
    }

    printf("Enter order of matrix: ");
    if (scanf("%d", &n) != 1 || n < 1 || n > MAX) {
        printf("Enter an order between 1 and %d.\n", MAX);
        close(sock);
        return 1;
    }

    srand((unsigned int)time(NULL));
    printf("\nGenerated Matrix:\n");

    for (int i = 0; i < n; i++) {
        for (int j = 0; j < n; j++) {
            matrix[i][j] = rand() % 50 + 1;
            printf("%4d", matrix[i][j]);
        }
        printf("\n");
    }

    if (!send_all(sock, &n, sizeof(n)) ||
        !send_all(sock, matrix, sizeof(matrix)) ||
        !receive_all(sock, result, sizeof(result))) {
        fprintf(stderr, "Communication with server failed.\n");
        close(sock);
        return 1;
    }

    printf("\nMatrix Type Received from Server: %s\n", result);
    close(sock);
    return 0;
}
```

------------------------------------------------------------------------

## Experiment 6 --- UDP Abbreviation Translator

### `udpserver.c`

``` c
#include <stdio.h>
#include <string.h>
#include <ctype.h>
#include <unistd.h>
#include <arpa/inet.h>

#define PORT 8080
#define MAX 1024

void add_word(char *result, const char *word)
{
    size_t used = strlen(result);
    if (used < MAX - 1)
        snprintf(result + used, MAX - used, "%s ", word);
}

void translate(char *str)
{
    char temp[MAX], result[MAX] = "";
    const char *full[] = {
        "to be honest", "I don't care", "I guess", "to be fair",
        "at the moment", "in real life", "laughing out loud",
        "as soon as possible", "oh my God", "talk to you later",
        "I don't know", "never mind"
    };
    const char *short_form[] = {
        "tbh", "idc", "ig", "tbf", "atm", "irl",
        "lol", "asap", "omg", "ttyl", "idk", "nvm"
    };

    snprintf(temp, sizeof(temp), "%s", str);

    for (char *token = strtok(temp, " ,.!?\n"); token;
         token = strtok(NULL, " ,.!?\n")) {
        char word[50];
        snprintf(word, sizeof(word), "%s", token);

        for (int i = 0; word[i]; i++)
            word[i] = (char)tolower((unsigned char)word[i]);

        int found = 0;
        for (int i = 0; i < 12; i++) {
            if (strcmp(word, short_form[i]) == 0) {
                add_word(result, full[i]);
                found = 1;
                break;
            }
        }

        if (!found) add_word(result, token);
    }

    snprintf(str, MAX, "%s", result);
}

int main(void)
{
    int sockfd;
    char buffer[MAX];
    struct sockaddr_in server = {0}, client = {0};
    socklen_t len = sizeof(client);

    sockfd = socket(AF_INET, SOCK_DGRAM, 0);
    if (sockfd < 0) { perror("socket"); return 1; }

    server.sin_family = AF_INET;
    server.sin_addr.s_addr = INADDR_ANY;
    server.sin_port = htons(PORT);

    if (bind(sockfd, (struct sockaddr *)&server, sizeof(server)) < 0) {
        perror("bind");
        close(sockfd);
        return 1;
    }

    printf("UDP Server Running...\n");

    ssize_t n = recvfrom(sockfd, buffer, MAX - 1, 0,
                         (struct sockaddr *)&client, &len);
    if (n >= 0) {
        buffer[n] = '\0';
        printf("Received: %s\n", buffer);
        translate(buffer);
        sendto(sockfd, buffer, strlen(buffer) + 1, 0,
               (struct sockaddr *)&client, len);
        printf("Translated message sent.\n");
    }

    close(sockfd);
    return 0;
}
```

### `udpclient.c`

``` c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

#define PORT 8080
#define MAX 1024

int main(void)
{
    int sockfd;
    char message[MAX], buffer[MAX];
    struct sockaddr_in server = {0};
    socklen_t len = sizeof(server);

    sockfd = socket(AF_INET, SOCK_DGRAM, 0);
    if (sockfd < 0) { perror("socket"); return 1; }

    server.sin_family = AF_INET;
    server.sin_port = htons(PORT);
    inet_pton(AF_INET, "127.0.0.1", &server.sin_addr);

    printf("Enter sentence:\n");
    if (!fgets(message, sizeof(message), stdin)) {
        close(sockfd);
        return 1;
    }

    if (sendto(sockfd, message, strlen(message) + 1, 0,
               (struct sockaddr *)&server, len) < 0) {
        perror("sendto");
        close(sockfd);
        return 1;
    }

    ssize_t n = recvfrom(sockfd, buffer, MAX - 1, 0,
                         (struct sockaddr *)&server, &len);
    if (n < 0) {
        perror("recvfrom");
        close(sockfd);
        return 1;
    }

    buffer[n] = '\0';
    printf("\nTranslated Sentence:\n%s\n", buffer);
    close(sockfd);
    return 0;
}
```

------------------------------------------------------------------------

## Experiment 7 --- Multi-user TCP Chat Server

### `chatserver.c`

``` c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <pthread.h>

#define PORT 8080
#define MAX_CLIENTS 10
#define BUFFER_SIZE 1024

int clients[MAX_CLIENTS];
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;

void broadcast(const char *message, int sender)
{
    pthread_mutex_lock(&lock);

    for (int i = 0; i < MAX_CLIENTS; i++) {
        if (clients[i] != -1 && clients[i] != sender)
            send(clients[i], message, strlen(message), 0);
    }

    pthread_mutex_unlock(&lock);
}

void *handle_client(void *arg)
{
    int client_socket = *(int *)arg;
    free(arg);

    char buffer[BUFFER_SIZE];
    ssize_t n;

    while ((n = recv(client_socket, buffer, BUFFER_SIZE - 1, 0)) > 0) {
        buffer[n] = '\0';
        broadcast(buffer, client_socket);
    }

    pthread_mutex_lock(&lock);
    for (int i = 0; i < MAX_CLIENTS; i++) {
        if (clients[i] == client_socket) {
            clients[i] = -1;
            break;
        }
    }
    pthread_mutex_unlock(&lock);

    close(client_socket);
    return NULL;
}

int main(void)
{
    int server_fd;
    struct sockaddr_in server = {0}, client = {0};
    socklen_t len = sizeof(client);

    for (int i = 0; i < MAX_CLIENTS; i++)
        clients[i] = -1;

    server_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (server_fd < 0) { perror("socket"); return 1; }

    server.sin_family = AF_INET;
    server.sin_addr.s_addr = INADDR_ANY;
    server.sin_port = htons(PORT);

    if (bind(server_fd, (struct sockaddr *)&server, sizeof(server)) < 0 ||
        listen(server_fd, 5) < 0) {
        perror("bind/listen");
        close(server_fd);
        return 1;
    }

    printf("Server running on port %d\n", PORT);

    while (1) {
        int client_socket = accept(server_fd, (struct sockaddr *)&client, &len);
        if (client_socket < 0) { perror("accept"); continue; }

        pthread_mutex_lock(&lock);
        int slot;
        for (slot = 0; slot < MAX_CLIENTS; slot++)
            if (clients[slot] == -1) break;

        if (slot == MAX_CLIENTS) {
            pthread_mutex_unlock(&lock);
            send(client_socket, "Server full\n", 12, 0);
            close(client_socket);
            continue;
        }

        clients[slot] = client_socket;
        pthread_mutex_unlock(&lock);

        int *arg = malloc(sizeof(int));
        if (!arg) {
            pthread_mutex_lock(&lock);
            clients[slot] = -1;
            pthread_mutex_unlock(&lock);
            close(client_socket);
            continue;
        }

        *arg = client_socket;
        pthread_t tid;
        if (pthread_create(&tid, NULL, handle_client, arg) != 0) {
            free(arg);
            pthread_mutex_lock(&lock);
            clients[slot] = -1;
            pthread_mutex_unlock(&lock);
            close(client_socket);
        } else {
            pthread_detach(tid);
        }
    }

    close(server_fd);
    return 0;
}
```

### `chatclient.c`

``` c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <pthread.h>

#define PORT 8080
#define BUFFER_SIZE 1024

int sockfd;

void *receive_msg(void *arg)
{
    (void)arg;
    char message[BUFFER_SIZE];
    ssize_t n;

    while ((n = recv(sockfd, message, BUFFER_SIZE - 1, 0)) > 0) {
        message[n] = '\0';
        printf("%s", message);
        fflush(stdout);
    }

    return NULL;
}

int main(void)
{
    struct sockaddr_in server = {0};
    char message[BUFFER_SIZE];

    sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sockfd < 0) { perror("socket"); return 1; }

    server.sin_family = AF_INET;
    server.sin_port = htons(PORT);
    inet_pton(AF_INET, "127.0.0.1", &server.sin_addr);

    if (connect(sockfd, (struct sockaddr *)&server, sizeof(server)) < 0) {
        perror("connect");
        close(sockfd);
        return 1;
    }

    pthread_t tid;
    if (pthread_create(&tid, NULL, receive_msg, NULL) != 0) {
        perror("pthread_create");
        close(sockfd);
        return 1;
    }

    while (fgets(message, sizeof(message), stdin)) {
        if (send(sockfd, message, strlen(message), 0) <= 0)
            break;
    }

    shutdown(sockfd, SHUT_WR);
    pthread_join(tid, NULL);
    close(sockfd);
    return 0;
}
```

------------------------------------------------------------------------

## Experiment 8 --- UDP Time Server

### `timeserver.c`

``` c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <time.h>

#define PORT 8080
#define BUFFER_SIZE 1024

int main(void)
{
    int sockfd;
    char buffer[BUFFER_SIZE];
    struct sockaddr_in server = {0}, client = {0};
    socklen_t len = sizeof(client);

    sockfd = socket(AF_INET, SOCK_DGRAM, 0);
    if (sockfd < 0) { perror("socket"); return 1; }

    server.sin_family = AF_INET;
    server.sin_addr.s_addr = INADDR_ANY;
    server.sin_port = htons(PORT);

    if (bind(sockfd, (struct sockaddr *)&server, sizeof(server)) < 0) {
        perror("bind");
        close(sockfd);
        return 1;
    }

    printf("UDP Time Server Running...\n");

    while (1) {
        ssize_t n = recvfrom(sockfd, buffer, BUFFER_SIZE - 1, 0,
                             (struct sockaddr *)&client, &len);
        if (n < 0) continue;

        time_t current_time = time(NULL);
        snprintf(buffer, sizeof(buffer), "%s", ctime(&current_time));

        sendto(sockfd, buffer, strlen(buffer) + 1, 0,
               (struct sockaddr *)&client, len);
    }

    close(sockfd);
    return 0;
}
```

### `timeclient.c`

``` c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

#define PORT 8080
#define BUFFER_SIZE 1024

int main(void)
{
    int sockfd;
    char buffer[BUFFER_SIZE] = "TIME";
    struct sockaddr_in server = {0};
    socklen_t len = sizeof(server);

    sockfd = socket(AF_INET, SOCK_DGRAM, 0);
    if (sockfd < 0) { perror("socket"); return 1; }

    server.sin_family = AF_INET;
    server.sin_port = htons(PORT);
    inet_pton(AF_INET, "127.0.0.1", &server.sin_addr);

    if (sendto(sockfd, buffer, strlen(buffer) + 1, 0,
               (struct sockaddr *)&server, len) < 0) {
        perror("sendto");
        close(sockfd);
        return 1;
    }

    ssize_t n = recvfrom(sockfd, buffer, BUFFER_SIZE - 1, 0,
                         (struct sockaddr *)&server, &len);
    if (n < 0) {
        perror("recvfrom");
        close(sockfd);
        return 1;
    }

    buffer[n] = '\0';
    printf("Server Time: %s\n", buffer);
    close(sockfd);
    return 0;
}
```

------------------------------------------------------------------------

## Experiment 9 --- Concurrent TCP File Server

### `fileserver.c`

``` c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <signal.h>
#include <sys/wait.h>

#define PORT 8080
#define BUFFER_SIZE 1024

int send_all(int socket, const char *data, size_t size)
{
    size_t sent = 0;

    while (sent < size) {
        ssize_t n = send(socket, data + sent, size - sent, 0);
        if (n <= 0) return 0;
        sent += n;
    }

    return 1;
}

int main(void)
{
    int server_fd;
    struct sockaddr_in server = {0}, client = {0};
    socklen_t len = sizeof(client);

    signal(SIGCHLD, SIG_IGN);

    server_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (server_fd < 0) { perror("socket"); return 1; }

    server.sin_family = AF_INET;
    server.sin_addr.s_addr = INADDR_ANY;
    server.sin_port = htons(PORT);

    if (bind(server_fd, (struct sockaddr *)&server, sizeof(server)) < 0 ||
        listen(server_fd, 5) < 0) {
        perror("bind/listen");
        close(server_fd);
        return 1;
    }

    printf("File Server Running...\n");

    while (1) {
        int client_socket = accept(server_fd, (struct sockaddr *)&client, &len);
        if (client_socket < 0) { perror("accept"); continue; }

        pid_t pid = fork();

        if (pid == 0) {
            char filename[100] = {0}, buffer[BUFFER_SIZE];
            close(server_fd);

            ssize_t n = recv(client_socket, filename, sizeof(filename) - 1, 0);
            if (n > 0) {
                filename[n] = '\0';
                filename[strcspn(filename, "\r\n")] = '\0';

                snprintf(buffer, sizeof(buffer), "Server PID: %d\n\n", getpid());
                send_all(client_socket, buffer, strlen(buffer));

                FILE *fp = fopen(filename, "r");
                if (!fp) {
                    strcpy(buffer, "Requested file does not exist.\n");
                    send_all(client_socket, buffer, strlen(buffer));
                } else {
                    while (fgets(buffer, sizeof(buffer), fp))
                        if (!send_all(client_socket, buffer, strlen(buffer)))
                            break;
                    fclose(fp);
                }
            }

            close(client_socket);
            exit(0);
        }

        if (pid < 0)
            perror("fork");

        close(client_socket);
    }

    close(server_fd);
    return 0;
}
```

### `fileclient.c`

``` c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

#define PORT 8080
#define BUFFER_SIZE 1024

int main(void)
{
    int sockfd;
    struct sockaddr_in server = {0};
    char filename[100], buffer[BUFFER_SIZE];

    sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sockfd < 0) { perror("socket"); return 1; }

    server.sin_family = AF_INET;
    server.sin_port = htons(PORT);
    inet_pton(AF_INET, "127.0.0.1", &server.sin_addr);

    if (connect(sockfd, (struct sockaddr *)&server, sizeof(server)) < 0) {
        perror("connect");
        close(sockfd);
        return 1;
    }

    printf("Enter filename: ");
    if (scanf("%99s", filename) != 1) {
        close(sockfd);
        return 1;
    }

    if (send(sockfd, filename, strlen(filename) + 1, 0) < 0) {
        perror("send");
        close(sockfd);
        return 1;
    }

    printf("\nResponse from Server:\n\n");

    ssize_t n;
    while ((n = recv(sockfd, buffer, sizeof(buffer), 0)) > 0)
        fwrite(buffer, 1, (size_t)n, stdout);

    close(sockfd);
    return 0;
}
```

### `sample.txt`

``` text
welcome to networks lab
```
