#include "stdafx.h"
#include <iostream>
#include <cmath>
using namespace std;
#include "./gdal/gdal_priv.h"
#pragma comment(lib, "gdal_i.lib")

void deal(char* dstPath, GDALDataset *poSrc1DS, GDALDataset *poSrc2DS,int imgXlen,int imgYlen,int sizeX,int sizeY) {
​	GDALDataset *poDstDS;
​	float *bandR, *bandG, *bandB, *bandI, *bandH, *bandS,*band;
​	int  xNum, yNum,nXS,nYS;


	xNum = (imgXlen - 1) / sizeX + 1;
	yNum = (imgYlen - 1) / sizeY + 1;
	
	cout << "xNum:" << xNum << "   " << "yNum:" << yNum << endl;



	band = (float*)CPLMalloc(sizeX * sizeY * sizeof(float));
	bandR = (float*)CPLMalloc(sizeX * sizeY * sizeof(float));
	bandG = (float*)CPLMalloc(sizeX * sizeY * sizeof(float));
	bandB = (float*)CPLMalloc(sizeX * sizeY * sizeof(float));
	bandI = (float*)CPLMalloc(sizeX * sizeY * sizeof(float));
	bandH = (float*)CPLMalloc(sizeX * sizeY * sizeof(float));
	bandS = (float*)CPLMalloc(sizeX * sizeY * sizeof(float));
	
	//create
	poDstDS = GetGDALDriverManager()->GetDriverByName("GTiff")->Create(
		dstPath, imgXlen, imgYlen, 3, GDT_Byte, NULL);
	
	for (int i = 0; i < xNum; i++) {
		for (int j = 0; j < yNum; j++) {
			//read
			//cout << i*sizeX << " " << j*sizeY << endl;
			nXS = sizeX;
			nYS = sizeY;
			//如果最下面和最右边的块不够256，剩下多少读取多少
			if (i*sizeX+nXS > imgXlen)     //最下面的剩余块
				nXS = imgXlen-i*sizeX;
			if (j*sizeY + nYS > imgYlen)     //最右侧的剩余块
				nYS = imgYlen - j*sizeY;
	
			//读数据
			poSrc1DS->GetRasterBand(1)->RasterIO(GF_Read, i*sizeX, j*sizeY, nXS, nYS,
				bandR, nXS, nYS, GDT_Float32, 0, 0);
			poSrc1DS->GetRasterBand(2)->RasterIO(GF_Read, i*sizeX, j*sizeY, nXS, nYS,
				bandG, nXS, nYS, GDT_Float32, 0, 0);
			poSrc1DS->GetRasterBand(3)->RasterIO(GF_Read, i*sizeX, j*sizeY, nXS, nYS,
				bandB, nXS, nYS, GDT_Float32, 0, 0);
			poSrc2DS->GetRasterBand(1)->RasterIO(GF_Read, i*sizeX, j*sizeY, nXS, nYS,
				band, nXS, nYS, GDT_Float32, 0, 0);
	
			//deal
			for (int k = 0; k < nXS*nYS; k++)
			{
				bandH[k] = -sqrt(2.0f) / 6.0f*bandR[k] - sqrt(2.0f) / 6.0f*bandG[k] + sqrt(2.0f) / 3.0f*bandB[k];
				bandS[k] = 1.0f / sqrt(2.0f)*bandR[k] - 1 / sqrt(2.0f)*bandG[k];
	
				bandR[k] = band[k] - 1.0f / sqrt(2.0f)*bandH[k] + 1.0f / sqrt(2.0f)*bandS[k];
				bandG[k] = band[k] - 1.0f / sqrt(2.0f)*bandH[k] - 1.0f / sqrt(2.0f)*bandS[k];
				bandB[k] = band[k] + sqrt(2.0f)*bandH[k];
			}
	
			//fuse
			//cout << i*sizeX << " " << j*sizeY << endl;
			poDstDS->GetRasterBand(1)->RasterIO(GF_Write, i*sizeX, j*sizeY, nXS, nYS,
				bandR, nXS, nYS, GDT_Float32, 0, 0);
			poDstDS->GetRasterBand(2)->RasterIO(GF_Write, i*sizeX, j*sizeY, nXS, nYS,
				bandG, nXS, nYS, GDT_Float32, 0, 0);
			poDstDS->GetRasterBand(3)->RasterIO(GF_Write, i*sizeX, j*sizeY, nXS, nYS,
				bandB, nXS, nYS, GDT_Float32, 0, 0);
		}
		cout << i << endl;
	}
	
	CPLFree(bandR);
	CPLFree(bandG);
	CPLFree(bandB);
	CPLFree(bandI);
	CPLFree(bandH);
	CPLFree(bandS);
	CPLFree(band);
	
	GDALClose(poDstDS);
}

int main() {

	char* src1Path = "Mul_large.tif";
	char* src2Path = "Pan_large.tif";
	
	//char* src1Path = "American_MUL.bmp";
	//char* src2Path = "American_PAN.bmp";
	
	GDALAllRegister();
	
	GDALDataset *poSrc1DS, *poSrc2DS;
	int imgXlen, imgYlen;
	clock_t start, finish;
	double duration;
	
	// read
	poSrc1DS = (GDALDataset*)GDALOpenShared(src1Path, GA_ReadOnly);
	poSrc2DS = (GDALDataset*)GDALOpenShared(src2Path, GA_ReadOnly);
	
	imgXlen = poSrc1DS->GetRasterXSize();
	imgYlen = poSrc1DS->GetRasterYSize();
	
	cout << "imgXlen:" << imgXlen << "   imgYlen:" << imgYlen << endl;


​	
	start = clock();
	deal("fus1.tif", poSrc1DS, poSrc2DS, imgXlen, imgYlen, 256, 256);//256*256
	finish = clock();
	duration = (double)(finish - start) / CLOCKS_PER_SEC;
	cout << "time1:" << duration << endl;
	//
	start = clock();
	deal("Fus2.tif", poSrc1DS, poSrc2DS, imgXlen, imgYlen, imgXlen, 256);//imgXlen*256
	finish = clock();
	duration = (double)(finish - start) / CLOCKS_PER_SEC;
	cout << "time2:" << duration << endl;


	GDALClose(poSrc1DS);
	GDALClose(poSrc2DS);
	
	GDALAllRegister();


	return 0;

}