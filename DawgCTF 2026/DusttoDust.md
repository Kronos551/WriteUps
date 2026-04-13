# Dust to Dust
<img width="1107" height="182" alt="DustToDustCard" src="https://github.com/user-attachments/assets/cce02f4d-6e14-4521-a002-7a8f7d90e2b6" />

In this challenge, we're given a C file and an output text file. Usually you aren't given just a raw C file in a reversing challenge, but I'm not one to complain. Let's start by opening up the output:

<img width="1336" height="411" alt="DustToDust_Output" src="https://github.com/user-attachments/assets/f3d916f9-249a-4f5c-873d-d74d0426b101" />

This is the output for whatever happens in the C file, and it most likely used to be the flag. Let's read the C file, starting at main:

```c
int main() {
    int l, w;
    char** str = allocArray(&l, &w);
    if (str == NULL) {
        return 1;
    }

    str = compressArray(str, 1, &l, &w);
    //str = compressArray(str, 2, &l, &w);

    writeArray(str, l);
    freeArray(str, l);
    return 0;
}
```

When the code runs, it calls allocArray() on a string, then calls compressArray(), writeArray(), and freeArray(). Let's look at what all of these functions in order to see what they do:

#### allocArray()
```c
char** allocArray(int* l, int* w) {
    int MAX_LINE_LENGTH = 1024, line_count = 0, line_curr = 0;
    FILE *input, *output;
    char** arrstarr_yarharhar;
    char buffer[MAX_LINE_LENGTH];
    size_t char_count = 0;

    input = fopen("input.txt", "r");
    if (input == NULL) {
        printf("Error opening file input.txt\n");
        return NULL;
    }

    while (fgets(buffer, MAX_LINE_LENGTH, input) != NULL) {
        line_count++;
        size_t curr = strlen(buffer);
        if ((curr - 1) % 3 != 0) {
            int num = (curr - 1);
            fclose(input);
            printf("Line %i does not have multiple of 3 characters, holds %i", line_count, num);
            return NULL;
        }
        if ((char_count != curr) && (char_count != 0)) {
            fclose(input);
            printf("Line %i does not match first line", line_count);
            return NULL;
        }
        if (strIsBin(buffer) == 0) {
            fclose(input);
            printf("Line %i does not contain only binary characters", line_count);
            return NULL;
        }
        if (char_count == 0) {
            char_count = curr;
        }
    }

    if (line_count % 2 != 0) {
        fclose(input);
        printf("File does not have a multiple of 2 lines; contains %i", line_count);
        return NULL;
    }


    
    arrstarr_yarharhar = malloc(line_count * sizeof (char *));
    if (arrstarr_yarharhar == NULL) {
        printf("malloc failed for char* array");
        return NULL;
    }

    fclose(input);
    input = fopen("input.txt", "r");
    
    while (fgets(buffer, MAX_LINE_LENGTH, input) != NULL) {
        arrstarr_yarharhar[line_curr] = malloc(char_count * sizeof(char));
        if (arrstarr_yarharhar[line_curr] == NULL) {
            printf("malloc failed at line %i", line_curr);
            return NULL;
        }

        sprintf(arrstarr_yarharhar[line_curr], buffer);
        line_curr++;
    }

    fclose(input);

    *l = line_count;
    *w = char_count;
    return arrstarr_yarharhar;
}
```

This is by far the longest function in the program, but not a lot of actual important computations happen at this step. The function opens an input file and then makes sure the contents of the file meet the following criteria:
+ Only binary characters are present (0 and 1)
+ There is an even amount of lines
+ The amount of characters in each line is divisible by 3.
If the input file meets this criteria, it allocates an array to store each line of the file, then for each line allocates an array to store each character. This essentially just creates a 2D array.

#### compressArray()
```c
char** compressArray(char** arr, int level, int* length, int* width) {
    if (*length % 2 != 0 && *width-1% 3 != 0) {
        printf("Array of size %ix%i cannot be compressed (is it not null terminated?)", length, width);
        return NULL;
    }

    int length_new = *length / 2;
    int width_new = (*width / 3) + 1;

    char** arr_new = malloc(length_new * sizeof (char*));
    for (int i = 0; i < length_new; i++) {
        arr_new[i] = malloc(width_new * sizeof(char));
    }

    for (int l = 0; l < length_new; l++) {
        for (int w = 0; w < width_new; w++) {
            if (w == width_new - 1) {
                arr_new[l][w] = '\0';
            }
            else {
                char buffer[7], c;

                if (level == 1) {

                    buffer[0] = arr[l*2][w*3];
                    buffer[1] = arr[l*2][w*3 + 1];
                    buffer[2] = arr[l*2][w*3 + 2];
                    buffer[3] = arr[l*2 + 1][w*3];
                    buffer[4] = arr[l*2 + 1][w*3 + 1];
                    buffer[5] = arr[l*2 + 1][w*3 + 2];
                    buffer[6] = '\0';

                    long bin = strtol(buffer, NULL, 2);
                    c = (char)(0b00100000 + bin);
                    arr_new[l][w] = c;
                }

                /*else if (level == 2) {
                    Ignore this part I'll add it later
                }*/
            }
        }

    }

    
    freeArray(arr, (length_new * 2));
    *length = length_new;
    *width = width_new;
    return arr_new;

}
```
This looks like some actually important computation. This function takes the 2D array and compresses it. It does this by taking a 2x3 chunk of characters, puts them into a string, converts that string into a long data type, adds 32, then converts that long to a single character. This character is then added to a new 2D array. At the end of the function, it calls the freeArray() function on the memory held by the original array, then returns the new array.

#### writeArray()
```c
void writeArray(char** arr, int len) {
    FILE* output = fopen("output.txt", "w");
    int i = 0;
    
    while (i < len) {
        fprintf(output, "%s%c", arr[i], (char)0b01111101);
        i++;
    }
    fprintf(output, "%c", (char)0b01111110);
    fclose(output);
}
```
Thankfully, this function is a lot shorter. It loops through the new array and writes the contents to an output file. At the end of every line, it uses the '}' character as a delimiter, and then uses '~' as a terminator at the end of the file.

#### freeArray()
```c
void freeArray(char** arr, int len) {
    for (int i = 0; i < len; i++) {
        free(arr[i]);
        arr[i] = NULL;
    }
    free(arr);
    arr = NULL;
}
```
This is a super simple function, all it does is free up the memory contained in an array line-by-line, just some best practice progamming stuff.


Now I know what the program does, and I know what its output is, but I still want to know what the output file used to be. We can see a comment in the program where the author "forgot to write" a level 2 to this function, likely supposed to be the unpacker they mention in the challenge description. This is probably where you're intended to write your solution script, but I lowkey really didn't feel like remembering how to code in C, so I just wrote it in python in a separate file:

```python
file = open('output.txt','r')
output = file.read()

# Delimit with } and remove ~ EOF
lines = output.split('}')
for line in lines:
    if "~" in line:
        lines = [line.replace("~", "") for line in lines]

# Do the math shit
for line in lines:
    topBucket = []
    bottomBucket = []
    for c in line:
        val = ord(c) - 32
        bits = f"{val:06b}"
        top, bottom = bits[:3], bits[3:]
        topBucket.append(top)
        bottomBucket.append(bottom)
    top_str = "".join(topBucket).replace("0", "█").replace("1", " ")
    bottom_str = "".join(bottomBucket).replace("0", "█").replace("1", " ")
    print(top_str)
    print(bottom_str)
```
