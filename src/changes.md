# Git Commit Sequence for Multithreaded Web Server Implementation

This document outlines a sequence of small, logical Git commits to implement a multithreaded web server with synchronization, scheduling policies, and security measures.

## Commit 1: Add basic data structures for request buffer

```c
// Define request structure
typedef struct {
    int fd;                  // Client socket descriptor
    char filename[MAXBUF];   // Requested filename
    int filesize;            // Size of requested file
    time_t arrival_time;     // Time when request arrived (for starvation prevention)
} request_t;

// Buffer to store requests
request_t *request_buffer = NULL;

// Synchronization primitives
pthread_mutex_t buffer_lock = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t buffer_not_empty = PTHREAD_COND_INITIALIZER;
pthread_cond_t buffer_not_full = PTHREAD_COND_INITIALIZER;

// Variables to track buffer state
int buffer_head = 0;
int buffer_tail = 0;

// Counter for starvation prevention
int starvation_threshold = 10; // Requests waiting longer than this many cycles get priority
```

**Commit message:** "Add data structures for request buffer and thread synchronization"

## Commit 2: Implement buffer initialization function

```c
// Initialize buffer
void buffer_init() {
    request_buffer = (request_t *)malloc(buffer_max_size * sizeof(request_t));
    if (!request_buffer) {
        fprintf(stderr, "Failed to allocate memory for request buffer\n");
        exit(1);
    }
    
    // Initialize random number generator for random scheduling
    srand(time(NULL));
}
```

**Commit message:** "Add buffer initialization function"

## Commit 3: Implement directory traversal prevention

```c
// Check if path contains directory traversal attempt
int is_path_safe(char *path) {
    // Check for ".." which could be used for directory traversal
    if (strstr(path, "..") != NULL) {
        return 0; // Not safe
    }
    
    // Check for suspicious path patterns
    if (strstr(path, "/.") != NULL || strstr(path, "//") != NULL) {
        return 0; // Not safe
    }
    
    // Additional checks could be added here
    
    return 1; // Safe
}
```

**Commit message:** "Add security function to prevent directory traversal attacks"

## Commit 4: Implement buffer add request function

```c
// Add request to buffer
void buffer_add_request(int fd, char *filename, int filesize) {
    pthread_mutex_lock(&buffer_lock);
    
    // Wait if buffer is full
    while (buffer_size >= buffer_max_size) {
        pthread_cond_wait(&buffer_not_full, &buffer_lock);
    }
    
    // Add request to buffer
    request_buffer[buffer_tail].fd = fd;
    strcpy(request_buffer[buffer_tail].filename, filename);
    request_buffer[buffer_tail].filesize = filesize;
    request_buffer[buffer_tail].arrival_time = time(NULL);
    
    buffer_tail = (buffer_tail + 1) % buffer_max_size;
    buffer_size++;
    
    // Signal that buffer is not empty
    pthread_cond_signal(&buffer_not_empty);
    pthread_mutex_unlock(&buffer_lock);
}
```

**Commit message:** "Implement function to add requests to the buffer with synchronization"

## Commit 5: Implement get next request index function for scheduling

```c
// Get next request based on scheduling policy
int get_next_request_index() {
    int i, next_index = -1;
    
    switch (scheduling_algo) {
        case 0: // FIFO
            // Always use head of buffer
            next_index = buffer_head;
            break;
            
        case 1: // Smallest-file-first (SFF)
            {
                int smallest_size = INT_MAX;
                time_t oldest_request = 0;
                
                // First pass: check for starvation
                for (i = 0; i < buffer_size; i++) {
                    int idx = (buffer_head + i) % buffer_max_size;
                    time_t current_time = time(NULL);
                    
                    // If a request has been waiting too long, give it priority
                    if (difftime(current_time, request_buffer[idx].arrival_time) > starvation_threshold) {
                        if (oldest_request == 0 || request_buffer[idx].arrival_time < oldest_request) {
                            oldest_request = request_buffer[idx].arrival_time;
                            next_index = idx;
                        }
                    }
                }
                
                // If no starving requests, use SFF
                if (next_index == -1) {
                    for (i = 0; i < buffer_size; i++) {
                        int idx = (buffer_head + i) % buffer_max_size;
                        if (request_buffer[idx].filesize < smallest_size) {
                            smallest_size = request_buffer[idx].filesize;
                            next_index = idx;
                        }
                    }
                }
            }
            break;
            
        case 2: // Random
            {
                // Select a random request
                int random_offset = rand() % buffer_size;
                next_index = (buffer_head + random_offset) % buffer_max_size;
            }
            break;
            
        default:
            // Default to FIFO
            next_index = buffer_head;
            break;
    }
    
    return next_index;
}
```

**Commit message:** "Implement scheduling policies for request selection"

## Commit 6: Implement buffer get request function

```c
// Remove request from buffer
request_t buffer_get_request() {
    pthread_mutex_lock(&buffer_lock);
    
    // Wait if buffer is empty
    while (buffer_size <= 0) {
        pthread_cond_wait(&buffer_not_empty, &buffer_lock);
    }
    
    // Get next request based on scheduling policy
    int next_index = get_next_request_index();
    request_t request = request_buffer[next_index];
    
    // If not FIFO, we need to re-arrange the buffer
    if (next_index != buffer_head) {
        // Shift all requests between head and next_index
        request_t temp = request_buffer[next_index];
        int i = next_index;
        while (i != buffer_head) {
            int prev = (i - 1 + buffer_max_size) % buffer_max_size;
            request_buffer[i] = request_buffer[prev];
            i = prev;
        }
        request_buffer[buffer_head] = temp;
    }
    
    // Update head
    buffer_head = (buffer_head + 1) % buffer_max_size;
    buffer_size--;
    
    // Signal that buffer is not full
    pthread_cond_signal(&buffer_not_full);
    pthread_mutex_unlock(&buffer_lock);
    
    return request;
}
```

**Commit message:** "Implement function to get requests from buffer based on scheduling policy"

## Commit 7: Update request handling functions to use safer string functions

```c
void request_error(int fd, char *cause, char *errnum, char *shortmsg, char *longmsg) {
    char buf[MAXBUF], body[MAXBUF];
    
    // Create the body of error message first (have to know its length for header)
    snprintf(body, MAXBUF, 
            "<!doctype html>\r\n"
            "<head>\r\n"
            "  <title>CYB-3053 WebServer Error</title>\r\n"
            "</head>\r\n"
            "<body>\r\n"
            "  <h2>%s: %s</h2>\r\n" 
            "  <p>%s: %s</p>\r\n"
            "</body>\r\n"
            "</html>\r\n", errnum, shortmsg, longmsg, cause);
    
    // Write out the header information for this response
    snprintf(buf, MAXBUF, "HTTP/1.0 %s %s\r\n", errnum, shortmsg);
    write_or_die(fd, buf, strlen(buf));
    
    snprintf(buf, MAXBUF, "Content-Type: text/html\r\n");
    write_or_die(fd, buf, strlen(buf));
    
    snprintf(buf, MAXBUF, "Content-Length: %lu\r\n\r\n", strlen(body));
    write_or_die(fd, buf, strlen(buf));
    
    // Write out the body last
    write_or_die(fd, body, strlen(body));
    
    // close the socket connection
    close_or_die(fd);
}

void request_serve_static(int fd, char *filename, int filesize) {
    int srcfd;
    char *srcp, filetype[MAXBUF], buf[MAXBUF];
    
    request_get_filetype(filename, filetype);
    srcfd = open_or_die(filename, O_RDONLY, 0);
    
    // Rather than call read() to read the file into memory, 
    // which would require that we allocate a buffer, we memory-map the file
    srcp = mmap_or_die(0, filesize, PROT_READ, MAP_PRIVATE, srcfd, 0);
    close_or_die(srcfd);
    
    // Use snprintf instead of sprintf to avoid buffer overflow
    snprintf(buf, MAXBUF, 
            "HTTP/1.0 200 OK\r\n"
            "Server: OSTEP WebServer\r\n"
            "Content-Length: %d\r\n"
            "Content-Type: %s\r\n\r\n", 
            filesize, filetype);
    
    write_or_die(fd, buf, strlen(buf));
    
    //  Writes out to the client socket the memory-mapped file 
    write_or_die(fd, srcp, filesize);
    munmap_or_die(srcp, filesize);
    
    // Close the socket connection
    close_or_die(fd);
}
```

**Commit message:** "Replace sprintf with snprintf for safer string handling"

## Commit 8: Implement thread request serve static function

```c
//
// Fetches the requests from the buffer and handles them (thread logic)
//
void* thread_request_serve_static(void* arg)
{
    // Initialize the buffer if it hasn't been initialized
    static int initialized = 0;
    if (!initialized) {
        pthread_mutex_lock(&buffer_lock);
        if (!initialized) {
            buffer_init();
            initialized = 1;
        }
        pthread_mutex_unlock(&buffer_lock);
    }
    
    // Thread main loop
    while (1) {
        // Get a request from the buffer
        request_t request = buffer_get_request();
        
        // Process the request
        request_serve_static(request.fd, request.filename, request.filesize);
    }
    
    return NULL;
}
```

**Commit message:** "Implement thread function to process requests from buffer"

## Commit 9: Update request_handle to use buffer and security checks

```c
//
// Initial handling of the request
//
void request_handle(int fd) {
    int is_static;
    struct stat sbuf;
    char buf[MAXBUF], method[MAXBUF], uri[MAXBUF], version[MAXBUF];
    char filename[MAXBUF], cgiargs[MAXBUF];
    
    // get the request type, file path and HTTP version
    readline_or_die(fd, buf, MAXBUF);
    sscanf(buf, "%s %s %s", method, uri, version);
    printf("method:%s uri:%s version:%s\n", method, uri, version);

    // verify if the request type is GET or not
    if (strcasecmp(method, "GET")) {
        request_error(fd, method, "501", "Not Implemented", "server does not implement this method");
        return;
    }
    request_read_headers(fd);
    
    // check requested content type (static/dynamic)
    is_static = request_parse_uri(uri, filename, cgiargs);
    
    // Security check to prevent directory traversal attacks
    if (!is_path_safe(filename)) {
        request_error(fd, filename, "403", "Forbidden", "directory traversal attempt detected");
        return;
    }
    
    // get some data regarding the requested file, also check if requested file is present on server
    if (stat(filename, &sbuf) < 0) {
        request_error(fd, filename, "404", "Not found", "server could not find this file");
        return;
    }
    
    // verify if requested content is static
    if (is_static) {
        if (!(S_ISREG(sbuf.st_mode)) || !(S_IRUSR & sbuf.st_mode)) {
            request_error(fd, filename, "403", "Forbidden", "server could not read this file");
            return;
        }
        
        // Add request to buffer
        buffer_add_request(fd, filename, sbuf.st_size);

    } else {
        request_error(fd, filename, "501", "Not Implemented", "server does not serve dynamic content request");
    }
}
```

**Commit message:** "Update request handler to use buffer and add security check"

## Commit 10: Add necessary include files and global variables

```c
#include <stdlib.h>
#include <time.h>
#include <string.h>
#include <limits.h>

// Global variables for buffer and synchronization
int buffer_max_size = DEFAULT_BUFFER_SIZE;
int buffer_size = 0;
int scheduling_algo = DEFAULT_SCHED_ALGO;
int num_threads = DEFAULT_THREADS;
```

**Commit message:** "Add necessary includes and global variables"