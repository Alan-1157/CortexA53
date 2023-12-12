# 基于CEC6818的图片编辑软件
基于CEC6818的平台的图片编辑软件，具有涂鸦图片、拼接、画笔选择功能

快速使用：
1.使用arm-linux-gcc交叉编译出可执行文件移植到arm板上
2.将album data的资源文件中的third_background.bmp和first.bmp放到与可执行目录同文件夹下
3.将图片资源放到./album目录下 ./是与当前可执行文件路径
4.执行./work ./album即可

主要功能
1.涂鸦 2.保存
draw.h
void draw_point(int x0, int y0, int color);
void draw_circle(int x0, int y0, int r, int color);
void save_screen_bmp(void* screen_data,const char* filename, int width, int height);

draw.c
void draw_point(int x0, int y0, int color)
{
    if (x0 >= 0 && x0 < 800 && y0 >= 0 && y0 < 480)
    {

        printf("draw point x=%d,y=%d\n", x0, y0);
        *(p + y0 * 800 + x0) = color;
    }
}

void draw_circle(int x0, int y0, int r, int color)
{
    int x = r;
    int y = 0;
    int err = 0;

    while (x >= y)
    {
        draw_point(x0 + x, y0 + y, color);
        draw_point(x0 + y, y0 + x, color);
        draw_point(x0 - y, y0 + x, color);
        draw_point(x0 - x, y0 + y, color);
        draw_point(x0 - x, y0 - y, color);
        draw_point(x0 - y, y0 - x, color);
        draw_point(x0 + y, y0 - x, color);
        draw_point(x0 + x, y0 - y, color);

        if (err <= 0)
        {
            y += 1;
            err += 2 * y + 1;
        }

        if (err > 0)
        {
            x -= 1;
            err -= 2 * x + 1;
        }
    }
    draw_point(x0, y0, color);
}

void save_screen_bmp(void* screen_data,const char* filename, int width, int height)
{
    FILE* file = fopen(filename, "wb+");
    if (file == NULL)
    {
        printf("Failed to open file for writing.\n");
        return;
    }

    // 设置位图文件??
    unsigned char bmp_file_header[14] = {
        'B', 'M', // 文件类型
        0, 0, 0, 0, // 文件大小（先??0??
        0, 0, // 保留字???
        0, 0, // 保留字???
        54, 0, 0, 0 // 数据偏移??
    };
    int file_size = 54 + width * height * 3;
    bmp_file_header[2] = (unsigned char)(file_size);
    bmp_file_header[3] = (unsigned char)(file_size >> 8);
    bmp_file_header[4] = (unsigned char)(file_size >> 16);
    bmp_file_header[5] = (unsigned char)(file_size >> 24);
    fwrite(bmp_file_header, sizeof(bmp_file_header), 1, file);

    // 设置位图信息??
    unsigned char bmp_info_header[40] = {
        40, 0, 0, 0, 
        0, 0, 0, 0, 
        0, 0, 0, 0, 
        1, 0, 
        24, 0, 
        0, 0, 0, 0,
        0, 0, 0, 0, 
        0, 0, 0, 0, 
        0, 0, 0, 0,
        0, 0, 0, 0, 
        0, 0, 0, 0 
    };
    bmp_info_header[4] = (unsigned char)(width);
    bmp_info_header[5] = (unsigned char)(width >> 8);
    bmp_info_header[6] = (unsigned char)(width >> 16);
    bmp_info_header[7] = (unsigned char)(width >> 24);
    bmp_info_header[8] = (unsigned char)(height);
    bmp_info_header[9] = (unsigned char)(height >> 8);
    bmp_info_header[10] = (unsigned char)(height >> 16);
    bmp_info_header[11] = (unsigned char)(height >> 24);
    int data_size = width * height * 3;
    bmp_info_header[20] = (unsigned char)(data_size);
    bmp_info_header[21] = (unsigned char)(data_size >> 8);
    bmp_info_header[22] = (unsigned char)(data_size >> 16);
    bmp_info_header[23] = (unsigned char)(data_size >> 24);
    fwrite(bmp_info_header, sizeof(bmp_info_header), 1, file);

    // 写入位图数据
    for (int y = height - 1; y >= 0; y--)
    {
        for (int x = 0; x < width; x++)
        {
            unsigned char* pixel = (unsigned char*)screen_data + y * width * 4 + x * 4;
            unsigned char b = pixel[0];
            unsigned char g = pixel[1];
            unsigned char r = pixel[2];
            unsigned char bmp_pixel[3] = { b, g, r };
            fwrite(bmp_pixel, sizeof(bmp_pixel), 1, file);
        }
    }

    fclose(file);
}


3.拼接
split.h

void combine_images(const char* image1_path, const char* image2_path, const char* output_path);
void handle_split(char *current_bmp_name,int end_x,int end_y);

split.c

void combine_images(const char* image1_path, const char* image2_path, const char* output_path) {
    FILE* image1_file = fopen(image1_path, "rb");
    FILE* image2_file = fopen(image2_path, "rb");
    FILE* output_file = fopen(output_path, "wb");

    if (image1_file ==NULL) {
        printf("Failed to open image1_file!\n");
        return;
    }
     if (image2_file ==NULL) {
        printf("Failed to open image2_file!\n");
        return;
    }
     if (output_file ==NULL) {
        printf("Failed to open output_file!\n");
        return;
    }

    BMPHeader image1_header, image2_header;
    BMPInfoHeader image1_info_header, image2_info_header;

    fread(&image1_header, sizeof(BMPHeader), 1, image1_file);
    fread(&image1_info_header, sizeof(BMPInfoHeader), 1, image1_file);
    fread(&image2_header, sizeof(BMPHeader), 1, image2_file);
    fread(&image2_info_header, sizeof(BMPInfoHeader), 1, image2_file);

    int width = image1_info_header.width;
    int height = image1_info_header.height;

    // 创建新的BMP文件头和信息头
    BMPHeader combined_header = image1_header;
    BMPInfoHeader combined_info_header = image1_info_header;
    combined_header.file_size += image2_header.file_size - image2_header.offset;
    combined_info_header.height *= 2;
    combined_info_header.image_size = combined_header.file_size - combined_header.offset;

    fwrite(&combined_header, sizeof(BMPHeader), 1, output_file);
    fwrite(&combined_info_header, sizeof(BMPInfoHeader), 1, output_file);

    // 复制第一张图片的像素数据
    fseek(image1_file, image1_header.offset, SEEK_SET);
    fseek(image2_file, image2_header.offset, SEEK_SET);
    fseek(output_file, combined_header.offset, SEEK_SET);

    unsigned char* buffer = (unsigned char*)malloc(width * 3);

    for (int y = 0; y < height; y++) {
        fread(buffer, width * 3, 1, image1_file);
        fwrite(buffer, width * 3, 1, output_file);
    }

    // 复制第二张图片的像素数据
    for (int y = 0; y < height; y++) {
        fread(buffer, width * 3, 1, image2_file);
        fwrite(buffer, width * 3, 1, output_file);
    }

    free(buffer);

    fclose(image1_file);
    fclose(image2_file);
    fclose(output_file);

    printf("Images combined successfully!\n");
}

void handle_split(char *current_bmp_name, int end_x, int end_y)
{
    printf("count=%d\n", *(p_data->p_count));
    char(*bmp_list)[4096] = (char(*)[4096])p_data->p;
    // 拼接 路径
    char dir[30];
    char fname[30];
    char ext[30];
    _splitpath(bmp_list[0], dir, fname, ext);

    char *out_path_small; // final image name
    // 展示当前图片小图

    for (int i = 0; i < *(p_data->p_count); i++)
    {
        char *out_pathname = malloc(50);
        srand(time(NULL));
        int n = rand() % 8;
        sprintf(out_pathname, "%sresult%d.bmp", dir, n);
        char *out_path_small_show = malloc(30);
        out_path_small = malloc(30); // final
        sprintf(out_path_small_show, "%smini_s%d.bmp", dir, i);
        sprintf(out_path_small, "%smini%d.bmp", dir, i);
        resizeBMP(bmp_list[i], out_path_small_show, 0.2);
        show_bmp_small(out_path_small_show, 0 + i * 150, 300);
        if ((0 + i * 150) < end_x && end_x < (160 + i * 150) && end_y > 300 && end_y < 396)
        {
            combine_images(current_bmp_name, bmp_list[i], out_pathname); // TODO 删除结合的大图片
            resizeBMP(out_pathname, out_path_small, 0.5);                // TODO，将缩小后的图片添加到bmp_name列表
            char *commod = malloc(50);
            sprintf(commod, "rm %s", out_pathname); // 删除结合后的大图
            system(commod);
            sprintf(commod, "rm ./album/mini_s*.bmp"); // 删除多选小图
            system(commod);
        }
    }

    *p_data->p_count = *(p_data->p_count) + 1;
    p_data->p = realloc(p_data->p, *p_data->p_count * 4096);
    strcpy((((char *)p_data->p) + ((*p_data->p_count - 1)) * 4096), out_path_small);
}
4.复原
复原即是再show一遍

还有resizeBMP和 spzlit_filepath功能函数对图像和路径进行处理不展示

清有收获的点个STAR收藏下

