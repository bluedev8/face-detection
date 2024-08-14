# libfacedetection

libfacedetection is a library for real-time face detection. This README provides detailed instructions on how to install the necessary tools, clone the repository, build the project, and compile it using Emscripten.

## Prerequisites

Before you begin, ensure you have the following tools installed on your system:

- Emscripten
- CMake

### Installing Emscripten

Follow the instructions from the official Emscripten documentation to install the latest version:

1. Clone the Emscripten SDK repository:
    ```sh
    git clone https://github.com/emscripten-core/emsdk.git
    ```

2. Navigate to the SDK directory:
    ```sh
    cd emsdk
    ```

3. Fetch the latest version of the Emscripten tools:
    ```sh
    ./emsdk install latest
    ```

4. Make the tools available in the current terminal session:
    ```sh
    ./emsdk activate latest
    source ./emsdk_env.sh
    ```

##Change the source code.

### Change the facedetectcnn.h source file.
Remove this
````
#include "facedetection_export.h"
````

Modify this
````
// FACEDETECTION_EXPORT int * facedetect_cnn(unsigned char * result_buffer, //buffer memory for storing face detection results, !!its size must be 0x20000 Bytes!!
//                    unsigned char * rgb_image_data, int width, int height, int step); //input image, it must be BGR (three channels) insteed of RGB image!

#ifdef __cplusplus

extern "C" {
#endif

int * facedetect_cnn(unsigned char * result_buffer, //buffer memory for storing face detection results, !!its size must be 0x20000 Bytes!!
                    unsigned char * rgb_image_data, int width, int height, int step); //input image, it must be BGR (three channels) insteed of RGB image!

#ifdef __cplusplus
}
#endif
````

###Change the facedetectcnn.cpp source file.

````
// CDataBlob<float> setDataFrom3x3S2P1to1x1S1P0FromImage(const unsigned char* inputData, int imgWidth, int imgHeight, int imgChannels, int imgWidthStep, int padDivisor) {
//     if (imgChannels != 3) {
//         std::cerr << __FUNCTION__ << ": The input image must be a 3-channel RGB image." << std::endl;
//         exit(1);
//     }
//     if (padDivisor != 32) {
//         std::cerr << __FUNCTION__ << ": This version need pad of 32" << std::endl;
//         exit(1);
//     }
//     int rows = ((imgHeight - 1) / padDivisor + 1) * padDivisor / 2;
//     int cols = ((imgWidth - 1) / padDivisor + 1 ) * padDivisor / 2;
//     int channels = 32;
//     CDataBlob<float> outBlob(rows, cols, channels);

// #if defined(_OPENMP)
// #pragma omp parallel for
// #endif
//     for (int r = 0; r < rows; r++) {
//         for (int c = 0; c < cols; c++) {
//             float* pData = outBlob.ptr(r, c);
//             for (int fy = -1; fy <= 1; fy++) {
//                 int srcy = r * 2 + fy;
                
//                 if (srcy < 0 || srcy >= imgHeight) //out of the range of the image
//                     continue;

//                 for (int fx = -1; fx <= 1; fx++) {
//                     int srcx = c * 2 + fx;

//                     if (srcx < 0 || srcx >= imgWidth) //out of the range of the image
//                         continue;

//                     const unsigned char * pImgData = inputData + size_t(imgWidthStep) * srcy + imgChannels * srcx;

//                     int output_channel_offset = ((fy + 1) * 3 + fx + 1) ; //3x3 filters, 3-channel image
//                     pData[output_channel_offset * imgChannels] = pImgData[0];
//                     pData[output_channel_offset * imgChannels + 1] = pImgData[1];
//                     pData[output_channel_offset * imgChannels + 2] = pImgData[2];
//                 }
//             }                    
//         }
//     } 
//     return outBlob;
// }

typedef struct {
    int start_row;
    int end_row;
    int num_cols;
    int imgWidth;
    int imgHeight;
    int imgChannels;
    int imgWidthStep;
    const unsigned char* inputData;
    CDataBlob<float>* outBlob;
} ThreadData_FromImage;

void* setDataFrom3x3S2P1to1x1S1P0FromImage_worker(void* arg) {
    ThreadData_FromImage* data = (ThreadData_FromImage*)arg;
    for (int r = data->start_row; r < data->end_row; r++) {
        for (int c = 0; c < data->num_cols; c++) {
            float* pData = data->outBlob->ptr(r, c);
            for (int fy = -1; fy <= 1; fy++) {
                int srcy = r * 2 + fy;

                if (srcy < 0 || srcy >= data->imgHeight) // out of the range of the image
                    continue;

                for (int fx = -1; fx <= 1; fx++) {
                    int srcx = c * 2 + fx;

                    if (srcx < 0 || srcx >= data->imgWidth) // out of the range of the image
                        continue;

                    const unsigned char* pImgData = data->inputData + size_t(data->imgWidthStep) * srcy + data->imgChannels * srcx;

                    int output_channel_offset = ((fy + 1) * 3 + fx + 1); // 3x3 filters, 3-channel image
                    pData[output_channel_offset * data->imgChannels] = pImgData[0];
                    pData[output_channel_offset * data->imgChannels + 1] = pImgData[1];
                    pData[output_channel_offset * data->imgChannels + 2] = pImgData[2];
                }
            }
        }
    }
    return NULL;
}

CDataBlob<float> setDataFrom3x3S2P1to1x1S1P0FromImage(const unsigned char* inputData, int imgWidth, int imgHeight, int imgChannels, int imgWidthStep, int padDivisor) {
    if (imgChannels != 3) {
        std::cerr << __FUNCTION__ << ": The input image must be a 3-channel RGB image." << std::endl;
        exit(1);
    }
    if (padDivisor != 32) {
        std::cerr << __FUNCTION__ << ": This version need pad of 32" << std::endl;
        exit(1);
    }

    int rows = ((imgHeight - 1) / padDivisor + 1) * padDivisor / 2;
    int cols = ((imgWidth - 1) / padDivisor + 1) * padDivisor / 2;
    int channels = 32;
    CDataBlob<float> outBlob(rows, cols, channels);

    int num_threads = 4;// emscripten_num_logical_cores(); // Get the number of logical cores
    if (num_threads <= 0) {
        std::cerr << __FUNCTION__ << ": Invalid number of threads." << std::endl;
        exit(1);
    }

    pthread_t threads[num_threads];
    ThreadData_FromImage thread_data[num_threads];

    int rows_per_thread = rows / num_threads;

    for (int i = 0; i < num_threads; i++) {
        thread_data[i].start_row = i * rows_per_thread;
        thread_data[i].end_row = (i == num_threads - 1) ? rows : (i + 1) * rows_per_thread;
        thread_data[i].num_cols = cols;
        thread_data[i].imgWidth = imgWidth;
        thread_data[i].imgHeight = imgHeight;
        thread_data[i].imgChannels = imgChannels;
        thread_data[i].imgWidthStep = imgWidthStep;
        thread_data[i].inputData = inputData;
        thread_data[i].outBlob = &outBlob;

        pthread_create(&threads[i], NULL, setDataFrom3x3S2P1to1x1S1P0FromImage_worker, &thread_data[i]);
    }

    for (int i = 0; i < num_threads; i++) {
        pthread_join(threads[i], NULL);
    }

    return outBlob;
}


typedef struct {
    int start_row;
    int end_row;
    int num_cols;
    int num_channels;
    CDataBlob<float>* outputData;
    int num_filters;  // Add this line to define num_filters
    const CDataBlob<float>* inputData;
    const Filters<float>* filters;
} ThreadData;

// bool convolution_1x1pointwise(const CDataBlob<float> & inputData, const Filters<float> & filters, CDataBlob<float> & outputData)
// {
// #if defined(_OPENMP)
// #pragma omp parallel for
// #endif
//     for (int row = 0; row < outputData.rows; row++)
//     {
//         for (int col = 0; col < outputData.cols; col++)
//         {
//             float * pOut = outputData.ptr(row, col);
//             const float * pIn = inputData.ptr(row, col);            
//             for (int ch = 0; ch < outputData.channels; ch++)
//             {
//                 const float * pF = filters.weights.ptr(0, ch);
//                 pOut[ch] = dotProduct(pIn, pF, inputData.channels);
//                 pOut[ch] += filters.biases.data[ch];
//             }
//         }
//     }
//     return true;
// }

void* convolution_1x1pointwise_worker(void* arg) {
    ThreadData* data = (ThreadData*)arg;
    for (int row = data->start_row; row < data->end_row; row++) {
        for (int col = 0; col < data->num_cols; col++) {
            float* pOut = data->outputData->ptr(row, col);
            const float* pIn = data->inputData->ptr(row, col);
            for (int ch = 0; ch < data->num_channels; ch++) {
                const float* pF = data->filters->weights.ptr(0, ch);
                pOut[ch] = dotProduct(pIn, pF, data->inputData->channels);
                pOut[ch] += data->filters->biases.data[ch];
            }
        }
    }
    return NULL;
}

bool convolution_1x1pointwise(const CDataBlob<float>& inputData, const Filters<float>& filters, CDataBlob<float>& outputData) {
    int num_threads = 4;//emscripten_num_logical_cores(); // Get the number of logical cores
    if (num_threads <= 0) {
        return false; // Ensure num_threads is valid
    }

    pthread_t threads[num_threads];
    ThreadData thread_data[num_threads];
    
    int rows_per_thread = outputData.rows / num_threads;
    
    EM_ASM({
        console.log("Num Threads:", $0);
    }, num_threads);

    for (int i = 0; i < num_threads; i++) {
        EM_ASM({
            console.log("Thread num:", $0);
        }, i);
        thread_data[i].start_row = i * rows_per_thread;
        thread_data[i].end_row = (i == num_threads - 1) ? outputData.rows : (i + 1) * rows_per_thread;
        thread_data[i].num_cols = outputData.cols;
        thread_data[i].num_channels = outputData.channels;
        thread_data[i].outputData = &outputData;
        thread_data[i].inputData = &inputData;
        thread_data[i].filters = &filters;
        
        pthread_create(&threads[i], NULL, convolution_1x1pointwise_worker, &thread_data[i]);
    }
    
    for (int i = 0; i < num_threads; i++) {
        EM_ASM({
            console.log("Thread num(join):", $0, $1);
        }, i, num_threads);
        pthread_join(threads[i], NULL);
    }
    
    return true;
}

// bool convolution_3x3depthwise(const CDataBlob<float> & inputData, const Filters<float> & filters, CDataBlob<float> & outputData)
// {
//     //set all elements in outputData to zeros
//     outputData.setZero();
// #if defined(_OPENMP)
// #pragma omp parallel for
// #endif
//     for (int row = 0; row < outputData.rows; row++) 
//     {  
//         int srcy_start = row - 1;
//         int srcy_end = srcy_start + 3;
//         srcy_start = MAX(0, srcy_start);
//         srcy_end = MIN(srcy_end, inputData.rows);

//         for (int col = 0; col < outputData.cols; col++)
//         { 
//             float * pOut = outputData.ptr(row, col);
//             int srcx_start = col - 1;
//             int srcx_end = srcx_start + 3;
//             srcx_start = MAX(0, srcx_start);
//             srcx_end = MIN(srcx_end, inputData.cols);


//             for ( int r = srcy_start; r < srcy_end; r++)
//                 for( int c = srcx_start; c < srcx_end; c++)
//                 {
//                     int filter_r = r - row + 1;
//                     int filter_c = c - col + 1;
//                     int filter_idx = filter_r * 3 + filter_c;
//                     vecMulAdd(inputData.ptr(r, c), filters.weights.ptr(0, filter_idx), pOut, filters.num_filters);
//                 }
//             vecAdd(filters.biases.ptr(0,0), pOut, filters.num_filters);
//         }
//     }
//      return true;
// }

void* convolution_3x3depthwise_worker(void* arg) {
    ThreadData* data = (ThreadData*)arg;
    for (int row = data->start_row; row < data->end_row; row++) {
        int srcy_start = row - 1;
        int srcy_end = srcy_start + 3;
        srcy_start = std::max(0, srcy_start);
        srcy_end = std::min(srcy_end, data->inputData->rows);

        for (int col = 0; col < data->num_cols; col++) {
            float* pOut = data->outputData->ptr(row, col);
            int srcx_start = col - 1;
            int srcx_end = srcx_start + 3;
            srcx_start = std::max(0, srcx_start);
            srcx_end = std::min(srcx_end, data->inputData->cols);

            for (int r = srcy_start; r < srcy_end; r++) {
                for (int c = srcx_start; c < srcx_end; c++) {
                    int filter_r = r - row + 1;
                    int filter_c = c - col + 1;
                    int filter_idx = filter_r * 3 + filter_c;
                    vecMulAdd(data->inputData->ptr(r, c), data->filters->weights.ptr(0, filter_idx), pOut, data->num_filters);
                }
            }
            vecAdd(data->filters->biases.ptr(0, 0), pOut, data->num_filters);
        }
    }
    return NULL;
}

bool convolution_3x3depthwise(const CDataBlob<float>& inputData, const Filters<float>& filters, CDataBlob<float>& outputData) {
    outputData.setZero();

    int num_threads = 4;//emscripten_num_logical_cores();
    if (num_threads <= 0) {
        return false;
    }

    pthread_t threads[num_threads];
    ThreadData thread_data[num_threads];

    int rows_per_thread = outputData.rows / num_threads;

    for (int i = 0; i < num_threads; i++) {
        thread_data[i].start_row = i * rows_per_thread;
        thread_data[i].end_row = (i == num_threads - 1) ? outputData.rows : (i + 1) * rows_per_thread;
        thread_data[i].num_cols = outputData.cols;
        thread_data[i].num_filters = filters.num_filters;  // Ensure this assignment
        thread_data[i].outputData = &outputData;
        thread_data[i].inputData = &inputData;
        thread_data[i].filters = &filters;

        pthread_create(&threads[i], NULL, convolution_3x3depthwise_worker, &thread_data[i]);
    }

    for (int i = 0; i < num_threads; i++) {
        pthread_join(threads[i], NULL);
    }

    return true;
}
````

## Compiling with Emscripten

Navigate to the `src` directory and run the following command to compile the project to WebAssembly:

```sh
em++ -O3 -s WASM=1 -msimd128 -s USE_PTHREADS=1 -s PTHREAD_POOL_SIZE=4 -s TOTAL_MEMORY=128MB ./facedetectcnn.cpp ./facedetectcnn-data.cpp ./facedetectcnn-model.cpp -o facedetection.js -s EXPORTED_FUNCTIONS=_malloc,_free,_facedetect_cnn
```

This will generate the facedetection.js and facedetection.wasm files.
