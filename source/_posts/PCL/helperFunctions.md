//create by luojinhong 2022 11 30
#pragma once
#define PCL_NO_PRECOMPILE

#include <pcl/point_cloud.h>
#include <pcl/point_types.h>
#include <pcl/conversions.h>
#include <pcl/common/centroid.h>
#include <pcl/common/common.h>
#include <pcl/common/transforms.h>

#include <pcl/kdtree/kdtree.h>
#include <pcl/search/kdtree.h> 
#include <pcl/filters/filter.h>
#include <pcl/filters/extract_indices.h>
#include <pcl/filters/voxel_grid.h>
#include <pcl/filters/crop_box.h>
#include <pcl/filters/crop_hull.h>
#include <pcl/filters/passthrough.h>

#include <pcl/registration/ndt.h>
#include <pcl/features/boundary.h>
#include <pcl/features/normal_3d_omp.h>
#include <pcl/features/normal_3d.h>
#include <pcl/features/moment_of_inertia_estimation.h>
#include <pcl/segmentation/sac_segmentation.h>
#include <pcl/segmentation/extract_clusters.h>
#include <pcl/features/fpfh.h>
#include <pcl/registration/ia_ransac.h>
#include <pcl/registration/icp.h>


#include <Eigen/Dense>
#include <vector>
#include <iostream> 
#include <string>  
#include <ctime>
#include <chrono>
#include <math.h>
#include <algorithm>

#include <Eigen/Dense>
#include <vector>



#include <opencv4/opencv2/highgui.hpp>   //计算凸包的最小包围盒
#include <opencv4/opencv2/imgproc.hpp>
#include <opencv4/opencv2/core/types.hpp>
#include <opencv4/opencv2/calib3d/calib3d_c.h>
#include <opencv4/opencv2/calib3d/calib3d.hpp>
#include <opencv4/opencv2/opencv.hpp>

void writePointCloudJH(const char* filename, pcl::PointCloud<pcl::PointXYZ>& cloud);
#pragma region imageProcess
struct MyRGB
{
	int r;
	int g;
	int b;
};

struct CamIntrinsic
{
    float fx;
    float fy;
    float cx;
    float cy;
    float k1;
    float k2;
    float p1;
    float p2;
    float k3;
};

struct PlaneMode
{
	PlaneMode():a(),b(),c(),d()
	{

	}
	PlaneMode(float _a, float _b,float _c,float _d)
	 :a(_a), b(_b),c(_c),d(_d){}
	float a = 0;
	float b = 0;
	float c = 1;
	float d = 0;

	//三点确定一个平面
	void getPlane(float x1,float y1,float z1
				,float x2,float y2,float z2
				,float x3,float y3,float z3)
	{
		float A = (y3 - y1)*(z3 - z1) - (z2 -z1)*(y3 - y1);
		float B = (x3 - x1)*(z2 - z1) - (x2 - x1)*(z3 - z1);
		float C = (x2 - x1)*(y3 - y1) - (x3 - x1)*(y2 - y1);
		float D = -(A * x1 + B * y1 + C * z1);
		a =A;
		b =B;
		c =C;
		d =D;
	}

	void transformPlane(Eigen::Matrix4f T)
	{
		Eigen::Vector4f newPlane = T*Eigen::Vector4f(a,b,c,d);
		a = newPlane(0);
		b = newPlane(1);
		c = newPlane(2);
		d = newPlane(3);
		
	}
	
	void transformPlane2(Eigen::Matrix4f T)
	{
		Eigen::Vector4f newPlane = T*Eigen::Vector4f(a,b,c,d);
		a = newPlane(0);
		b = newPlane(1);
		c = newPlane(2);
		d = newPlane(3);

		   //aX+cY+cZ+d =0;
    	pcl::PointXYZ ptO,ptA,ptB;
    	ptO.x = 0;
    	ptO.y = 0;
    	ptO.z = -d/c;
    	ptA.x = 1;
    	ptA.y = 0;
    	ptA.z = -(d+a*ptA.x)/c;
    	ptB.x = 0;
    	ptB.y = 1;
    	ptB.z = -(d+b*ptB.y)/c;
    	pcl::PointCloud<pcl::PointXYZ> clloudIn;
    	clloudIn.push_back(ptA);
    	clloudIn.push_back(ptB);
    	clloudIn.push_back(ptO);
    	pcl::PointCloud<pcl::PointXYZ> clloudOut;
    	pcl::transformPointCloud(clloudIn,clloudOut,T);
    	pcl::PointXYZ ptONew,ptANew,ptBNew;
    	ptONew = clloudOut.points[2];
    	ptBNew = clloudOut.points[1];
    	ptANew = clloudOut.points[0];
		getPlane(ptONew.x,ptONew.y,ptONew.z,
                            ptBNew.x,ptBNew.y,ptBNew.z,
                            ptANew.x,ptANew.y,ptANew.z);
	}

	pcl::PointXYZ getFootInPlane(const pcl::PointXYZ& pt)
	{
		float x0 = 0;
		float y0 = 0;
		float z0 = -d/c;
		float t = (a*x0+b*y0+c*z0 -(a*pt.x+b*pt.y+c*pt.z))/(a*a+b*b+c*c);
		pcl::PointXYZ foot;
		foot.x = pt.x + a*t;
		foot.y = pt.y + b*t;
		foot.z = pt.z + c*t;
		return foot;
	}
};

void EdgePoint_Trace(cv::Mat& edgeMag_noMaxsup, cv::Mat& edge, unsigned TL, int r, int c, int rows, int cols);
/**
 * @brief 区域生长算法，输入图像应为三通道图像（RGB、HSV、YUV等）
 * @param srcImage 区域生长的源图像
 * @param pt 区域生长点
 * @param ch1Thres ch2Thres ch3Thres 三个通道的生长限制阈值，临近像素符合±chxThres范围内才能进行生长
 * @param ch1LowerBind ch1LowerBind ch1LowerBind 三个通道的最小值阈值
 * @param ch1UpperBind ch2UpperBind ch3UpperBind 三个通道的最大值阈值，在这个范围外即使临近像素符合±chxThres也不能生长
 * @return 生成的区域图像（二值类型）
 */
Eigen::Matrix4f CreateRotateMatrix(Eigen::Vector3f before, Eigen::Vector3f after);
cv::Mat regionGrowSegmentation(cv::Mat srcImage, cv::Point pt, int ch1Thres,int ch2Thres, int ch3Thres,
               int ch1LowerBind,int ch1UpperBind,int ch2LowerBind,
               int ch2UpperBind,int ch3LowerBind,int ch3UpperBind);

/**
 * @brief 区域生长算法，输入图像应为灰度图像
 * @param srcImage 区域生长的源图像
 * @param pt 区域生长点
 * @param ch1Thres 通道的生长限制阈值，临近像素符合±chxThres范围内才能进行生长
 * @param ch1LowerBind 通道的最小值阈值
 * @param ch1UpperBind 通道的最大值阈值，在这个范围外即使临近像素符合±chxThres也不能生长
 * @return 生成的区域图像（二值类型）
 */
cv::Mat regionGrowSegmentation(cv::Mat srcImage, cv::Point pt, int ch1Thres,int ch1LowerBind,int ch1UpperBind);

void getPointCloudImage(pcl::PointCloud<pcl::PointXYZ>::Ptr cloud,cv::Mat& mask,const CamIntrinsic& intrinsic);
#pragma endregion

#pragma region AlgCommon


struct Color
{

	float r, g, b;

	Color(float setR, float setG, float setB)
		: r(setR), g(setG), b(setB)
	{}
};
//点云裁剪
struct CuttingBox
{
	Eigen::Vector4f minPoint;
	Eigen::Vector4f maxPoint;
};

struct ObjectOBBbox
{
	pcl::PointXYZ min_point;
	pcl::PointXYZ max_point;
	float width;
	float height;
	float depth;
	pcl::PointXYZ translation;
	Eigen::Matrix3f rotational_matrix;
};

struct ObjectAABBbox
{
	pcl::PointXYZ min_point;
	pcl::PointXYZ max_point;
};

struct ObjectXYHull
{
	pcl::PointCloud<pcl::PointXYZ> cloud;//该点运用于描述凸包形状
	std::vector<pcl::PointXYZ> vpts;//XY最小包围的顶点(4个顶点)

	float minZ = FLT_MAX;
	float maxZ = -FLT_MAX;
	ObjectXYHull()
	{
		vpts.resize(4);
		minZ = FLT_MAX;
		maxZ = -FLT_MAX;
	}

	//拷贝构造
	ObjectXYHull(const ObjectXYHull& other)
	{
		pcl::copyPointCloud(other.cloud, cloud);
		vpts = other.vpts;
		minZ = other.minZ;
		maxZ = other.maxZ;
	}
	//重载=
	ObjectXYHull& operator=(const ObjectXYHull& other)
	{
		pcl::copyPointCloud(other.cloud, cloud);
		vpts = other.vpts;
		minZ = other.minZ;
		maxZ = other.maxZ;
		return *this;
	}
};


//brief: 得到最小包围OBB盒和沿XYZ轴AABB盒
template<typename PointT>
inline void getBOXformCloud(const typename pcl::PointCloud<PointT>& cloudin, ObjectOBBbox& obb, ObjectAABBbox& aabb)
{
	PointT minaabb;
	PointT maxaabb;
	PointT minobb;
	PointT maxobb;
	PointT t;
	pcl::MomentOfInertiaEstimation <PointT> feature_extractor;
	feature_extractor.setInputCloud(cloudin.makeShared());
	feature_extractor.compute();
	feature_extractor.getAABB(minaabb, maxaabb);
	//将Z设置为0
	pcl::PointCloud<PointT> cloudNoZ;
	pcl::copyPointCloud(cloudin, cloudNoZ);
#pragma omp for
	for (int i = 0; i < cloudNoZ.size(); i++)
	{
		cloudNoZ.points[i].z = 0;
	}
	feature_extractor.setInputCloud(cloudNoZ.makeShared());
	feature_extractor.compute();
	feature_extractor.getOBB(minobb, maxobb, t, obb.rotational_matrix);
	pcl::copyPoint(minaabb, aabb.min_point);
	pcl::copyPoint(maxaabb, aabb.max_point);
	pcl::copyPoint(minobb, obb.min_point);
	pcl::copyPoint(maxobb, obb.max_point);
	pcl::copyPoint(t, obb.translation);
	obb.max_point.z = aabb.max_point.z;
	obb.min_point.z = aabb.min_point.z;
	obb.translation.z = (obb.max_point.z+ obb.min_point.z)/2;
	obb.width = obb.max_point.x - obb.min_point.x;
	obb.height = obb.max_point.y - obb.min_point.y;
	obb.depth = obb.max_point.z - obb.min_point.z;
}


template<typename PointT>
inline void getPointsInBox(const pcl::PointCloud<PointT>& cloudin, pcl::PointCloud<PointT>& cloudout, CuttingBox box)
{
	pcl::CropBox<PointT> roi;
	roi.setMin(box.minPoint);
	roi.setMax(box.maxPoint);
	roi.setInputCloud(cloudin.makeShared());
	roi.filter(cloudout);
}

template<typename PointT>
inline void removePointsInBox(const pcl::PointCloud<PointT>& cloudin, pcl::PointCloud<PointT>& cloudout, CuttingBox box)
{
	pcl::CropBox<PointT> roi;
	roi.setMin(box.minPoint);
	roi.setMax(box.maxPoint);
	roi.setNegative(true);
	roi.setInputCloud(cloudin.makeShared());
	roi.filter(cloudout);
}

Eigen::Matrix4f getCloudToSpecialVectorR(const pcl::PointCloud<pcl::PointXYZ> &cloud,Eigen::Vector3f after, int maxIteration, float distanThre,PlaneMode& planemodel);
Eigen::Matrix4f getCloudToFloorT(const pcl::PointCloud<pcl::PointXYZ>& cloud, float distanThre,PlaneMode& planemodel);


//std::vector<boost::filesystem::path> streamPcd(std::string dataPath)
//{
//
//	std::vector<boost::filesystem::path> paths(boost::filesystem::directory_iterator{ dataPath }, boost::filesystem::directory_iterator{});
//
//	// sort files in accending order so playback is chronological
//	sort(paths.begin(), paths.end());
//
//	return paths;
//
//}
void getFiles(const std::string& path, std::vector<std::string>& files);
void GetFileName(std::string path, std::vector<std::string> &files); 
void GetFileOnlyName(std::string path, std::vector<std::string> &files);

//
void getXYHullFromCloud(const pcl::PointCloud<pcl::PointXYZ>& cloud, ObjectXYHull& hullXY);

#pragma endregion

#pragma region ClusterRegion
struct PointCloudClusterParamsters
{
	float distanceTol;
	int minSize;
	int maxSize;
};

Eigen::Matrix4f CreateRotateMatrix(Eigen::Vector3f before, Eigen::Vector3f after);
template<typename PointT>
class PointCloudCluster
{
public:
	//main interface
	std::vector<typename pcl::PointCloud<PointT>::Ptr> euclideanClustering(typename pcl::PointCloud<PointT>::Ptr cloud, PointCloudClusterParamsters para);

private:

};

template<typename PointT>
inline std::vector<typename pcl::PointCloud<PointT>::Ptr> PointCloudCluster<PointT>::euclideanClustering(const typename pcl::PointCloud<PointT>::Ptr cloud, PointCloudClusterParamsters para)
{

	std::vector<typename pcl::PointCloud<PointT>::Ptr> clusters;

	// TODO:: Fill in the function to perform euclidean clustering to group detected obstacles
	// Creating the KdTree object for the search method of the extraction
	typename pcl::search::KdTree<PointT>::Ptr tree(new pcl::search::KdTree<PointT>);
	tree->setInputCloud(cloud);

	std::vector<pcl::PointIndices> cluster_indices;
	pcl::EuclideanClusterExtraction<PointT> ec;
	ec.setClusterTolerance(para.distanceTol); // 2cm
	ec.setMinClusterSize(para.minSize);
	ec.setMaxClusterSize(para.maxSize);
	ec.setSearchMethod(tree);
	ec.setInputCloud(cloud);
	ec.extract(cluster_indices);

	for (std::vector<pcl::PointIndices>::const_iterator it = cluster_indices.begin(); it != cluster_indices.end(); ++it)
	{
		typename pcl::PointCloud<PointT>::Ptr cloud_cluster(new pcl::PointCloud<PointT>);
		for (std::vector<int>::const_iterator pit = it->indices.begin(); pit != it->indices.end(); ++pit)
			cloud_cluster->points.push_back(cloud->points[*pit]); //*
		cloud_cluster->width = cloud_cluster->points.size();
		cloud_cluster->height = 1;
		cloud_cluster->is_dense = true;

		clusters.push_back(cloud_cluster);
	}

	return clusters;
}
#pragma endregion

//输入：平面坐标
//输入：多项式曲线的阶数（order）直线为1 抛物线为2 三次曲线为3
//输出：ret
bool FitterLeastSquareMethod(std::vector<float> &X, std::vector<float> &Y, int startIndex, int endIndex, uint8_t orders, Eigen::VectorXd& ret);


//平面直线参数
struct line2DTools
{
	line2DTools()
	{
		
	}
	line2DTools(float A,float B,float C):m_a(A),m_b(B),m_c(C)
	{

	}
	//a*X+b*y+c=0
	float m_a = NAN;
	float m_b = NAN;
	float m_c = NAN;

	//角度
	float m_angle;//角度
	cv::Point2f m_startPt;
	cv::Point2f m_endPt;
	//输入两个点获得直线
	bool checkLine()
	{
		if(m_a==NAN||m_b==NAN)
			return false;
		if(m_a==0&&m_b==0)
			return false;
		return true;
	}

	float gety(float x)
	{
		return -(m_a*x+m_c)/m_b;
	}
	float getx(float y)
	{
		return -(m_b*y+m_c)/m_a;
	}
	void getline(float line1X, float line1Y,float line2X,float line2Y)
	{
		m_a = line1Y - line2Y;
		m_b = line2X - line1X;
    	m_c = line1X*line2Y - line1Y*line2X;
		m_startPt = cv::Point2f(line1X,line1Y);
		m_endPt = cv::Point2f(line2X,line2Y);

	}
	float getLen()
	{
		return sqrt((m_startPt.x-m_endPt.x)*(m_startPt.x-m_endPt.x)+(m_startPt.y-m_endPt.y)*(m_startPt.y-m_endPt.y));
	}

	inline float getangle(Eigen::Vector3f before,Eigen::Vector3f after)
	{
        float signal = (after.x()*after.y())<0?1.0:-1.0;
        float alphaAngle = signal*acos(before.dot(after));
		return alphaAngle;
	}
	//点到直线的距离
	float getPoint2LineDis(float X,float Y)
	{
		if(!checkLine())
			return NAN;
		float distance = 0;
    	distance = (m_a*X + m_b*Y + m_c) /sqrtf(m_a*m_a + m_b*m_b);
		return distance;
	}
	//求垂足
	void getFootDown(float PtX,float PtY,float& footX,float& footY)
	{
		if(!checkLine())
		{ 
			footX = NAN;
			footY = NAN;
		}
		footX=(m_b*m_b*PtX-m_a*m_b*PtY-m_a*m_c)/(m_a*m_a+m_b*m_b);
        footY=(-m_a*m_b*PtX+m_a*m_a*PtY-m_b*m_c)/(m_a*m_a+m_b*m_b);
	}
};

bool fittingLine2D(std::vector<float> &X, std::vector<float> &Y, int startIndex, int endIndex,line2DTools& line);
bool fittingLine2DSVD(std::vector<float> &X, std::vector<float> &Y,int startIndex,int endIndex,line2DTools& line);

#include "../include/algCommon.h"
//自适应canny
//从确定边缘点出发，延长边缘
bool inline checkInRang(int r, int c, int rows, int cols) {
	if (r >= 0 && r < rows && c >= 0 && c < cols)
		return true;
	else
		return false;
}

void EdgePoint_Trace(cv::Mat& edgeMag_noMaxsup, cv::Mat& edge, unsigned TL, int r, int c, int rows, int cols)
{
	//如果边缘图未被标记
	if (edge.at<uchar>(r, c) == 0)
	{
		edge.at<uchar>(r, c) = 255;
		for (int i = -1; i <= 1; ++i)
		{
			for (int j = -1; j <= 1; ++j)
			{
				float mag = edgeMag_noMaxsup.at<float>(r + i, c + j);
				if (checkInRang(r + i, c + j, rows, cols) && mag >= TL)
					EdgePoint_Trace(edgeMag_noMaxsup, edge, TL, r + i, c + j, rows, cols);
			}
		}
	}
}

                                                                                                              
            

void writePointCloudJH(const char* filename, pcl::PointCloud<pcl::PointXYZ>& cloud)
{
    FILE *fp = NULL;

    unsigned int i;
    float distance_min=0.012f;
    int numPoints = cloud.size();

    fp = fopen(filename, "w"); //save more frames
    if(!fp)
    {
        printf(" [tofSensor]  Open %s error\n", (char *)filename);
        return;
    }
    fprintf(fp,"%s\n", "ply");
    fprintf(fp,"%s\n", "format ascii 1.0");
    fprintf(fp,"%s %d\n", "element vertex", numPoints);
    fprintf(fp,"%s\n", "property float32 x");
    fprintf(fp,"%s\n", "property float32 y");
    fprintf(fp,"%s\n", "property float32 z");
    fprintf(fp,"%s\n", "end_header");

    for (i = 0; i < numPoints; i ++)    //2020-3-22
    {

        if(sqrt(cloud[i].x*cloud[i].x + cloud[i].y*cloud[i].y +cloud[i].z*cloud[i].z) < distance_min )
        {
            fprintf(fp,"%f %f %f\n", 0.0, 0.0, 0.0);
        }
        else
        {
            fprintf(fp,"%f %f %f\n", cloud[i].x, cloud[i].y, cloud[i].z);
        }
    }

    fclose(fp);

    return;
}
// /**
//  * @brief 区域生长算法，输入图像应为灰度图像
//  * @param srcImage 区域生长的源图像
//  * @param pt 区域生长点
//  * @param ch1Thres 通道的生长限制阈值，临近像素符合±chxThres范围内才能进行生长
//  * @param ch1LowerBind 通道的最小值阈值
//  * @param ch1UpperBind 通道的最大值阈值，在这个范围外即使临近像素符合±chxThres也不能生长
//  * @return 生成的区域图像（二值类型）
//  */
cv::Mat regionGrowSegmentation(cv::Mat srcImage, cv::Point pt, int ch1Thres, int ch1LowerBind = 0, int ch1UpperBind = 255)
{
    cv::Point pToGrowing;                                         // 待生长点位置
    int pGrowValue = 0;                                           // 待生长点灰度值
    cv::Scalar pSrcValue = 0;                                     // 生长起点灰度值
    cv::Scalar pCurValue = 0;                                     // 当前生长点灰度值
    cv::Mat growImage = cv::Mat::zeros(srcImage.size(), CV_8UC1); // 创建一个空白区域，填充为黑色
    // 生长方向顺序数据
    int DIR[8][2] = {{-1, -1}, {0, -1}, {1, -1}, {1, 0}, {1, 1}, {0, 1}, {-1, 1}, {-1, 0}};
    std::vector<cv::Point> growPtVector;        // 生长点栈
    growPtVector.push_back(pt);                 // 将生长点压入栈中
    growImage.at<uchar>(pt.y, pt.x) = 255;      // 标记生长点
    pSrcValue = srcImage.at<uchar>(pt.y, pt.x); // 记录生长点的灰度值

    while (!growPtVector.empty()) // 生长栈不为空则生长
    {
        pt = growPtVector.back(); // 取出一个生长点
        growPtVector.pop_back();

        // 分别对八个方向上的点进行生长
        for (int i = 0; i < 8; ++i)
        {
            pToGrowing.x = pt.x + DIR[i][0];
            pToGrowing.y = pt.y + DIR[i][1];
            // 检查是否是边缘点
            if (pToGrowing.x < 0 || pToGrowing.y < 0 ||
                pToGrowing.x > (srcImage.cols - 1) || (pToGrowing.y > srcImage.rows - 1))
                continue;

            pGrowValue = growImage.at<uchar>(pToGrowing.y, pToGrowing.x); // 当前待生长点的灰度值
            pSrcValue = srcImage.at<uchar>(pt.y, pt.x);
            if (pGrowValue == 0) // 如果标记点还没有被生长
            {
                pCurValue = srcImage.at<uchar>(pToGrowing.y, pToGrowing.x);
                if (pCurValue[0] <= ch1UpperBind && pCurValue[0] >= ch1LowerBind)
                {
                    if (abs(pSrcValue[0] - pCurValue[0]) < ch1Thres) // 在阈值范围内则生长
                    {
                        growImage.at<uchar>(pToGrowing.y, pToGrowing.x) = 255; // 标记为白色
                        growPtVector.push_back(pToGrowing);                    // 将下一个生长点压入栈中
                    }
                }
            }
        }
    }
    return growImage.clone();
}

/**
 * @brief 区域生长算法，输入图像应为三通道图像（RGB、HSV、YUV等）
 * @param srcImage 区域生长的源图像
 * @param pt 区域生长点
 * @param ch1Thres ch2Thres ch3Thres 三个通道的生长限制阈值，临近像素符合±chxThres范围内才能进行生长
 * @param ch1LowerBind ch1LowerBind ch1LowerBind 三个通道的最小值阈值
 * @param ch1UpperBind ch2UpperBind ch3UpperBind 三个通道的最大值阈值，在这个范围外即使临近像素符合±chxThres也不能生长
 * @return 生成的区域图像（二值类型）
 */
cv::Mat regionGrowSegmentation(cv::Mat srcImage, cv::Point pt, int ch1Thres, int ch2Thres, int ch3Thres,
                               int ch1LowerBind = 0, int ch1UpperBind = 255, int ch2LowerBind = 0,
                               int ch2UpperBind = 255, int ch3LowerBind = 0, int ch3UpperBind = 255)
{
    cv::Point pToGrowing;                                         // 待生长点位置
    int pGrowValue = 0;                                           // 待生长点灰度值
    cv::Scalar pSrcValue = 0;                                     // 生长起点灰度值
    cv::Scalar pCurValue = 0;                                     // 当前生长点灰度值
    cv::Mat growImage = cv::Mat::zeros(srcImage.size(), CV_8UC1); // 创建一个空白区域，填充为黑色
    // 生长方向顺序数据
    int DIR[8][2] = {{-1, -1}, {0, -1}, {1, -1}, {1, 0}, {1, 1}, {0, 1}, {-1, 1}, {-1, 0}};
    std::vector<cv::Point> growPtVector;            // 生长点栈
    growPtVector.push_back(pt);                     // 将生长点压入栈中
    growImage.at<uchar>(pt.y, pt.x) = 255;          // 标记生长点
    pSrcValue = srcImage.at<cv::Vec3b>(pt.y, pt.x); // 记录生长点的灰度值

    while (!growPtVector.empty()) // 生长栈不为空则生长
    {
        pt = growPtVector.back(); // 取出一个生长点
        growPtVector.pop_back();

        // 分别对八个方向上的点进行生长
        for (int i = 0; i < 8; ++i)
        {
            pToGrowing.x = pt.x + DIR[i][0];
            pToGrowing.y = pt.y + DIR[i][1];
            // 检查是否是边缘点
            if (pToGrowing.x < 0 || pToGrowing.y < 0 ||
                pToGrowing.x > (srcImage.cols - 1) || (pToGrowing.y > srcImage.rows - 1))
                continue;

            pGrowValue = growImage.at<uchar>(pToGrowing.y, pToGrowing.x); // 当前待生长点的灰度值
            pSrcValue = srcImage.at<cv::Vec3b>(pt.y, pt.x);
            if (pGrowValue == 0) // 如果标记点还没有被生长
            {
                pCurValue = srcImage.at<cv::Vec3b>(pToGrowing.y, pToGrowing.x);
                if (pCurValue[0] <= ch1UpperBind && pCurValue[0] >= ch1LowerBind && // 限制生长点的三通道上下界
                    pCurValue[1] <= ch2UpperBind && pCurValue[1] >= ch2LowerBind &&
                    pCurValue[2] <= ch3UpperBind && pCurValue[2] >= ch3LowerBind)
                {
                    if (abs(pSrcValue[0] - pCurValue[0]) < ch1Thres &&
                        abs(pSrcValue[1] - pCurValue[1]) < ch2Thres &&
                        abs(pSrcValue[2] - pCurValue[2]) < ch3Thres) // 在阈值范围内则生长
                    {
                        growImage.at<uchar>(pToGrowing.y, pToGrowing.x) = 255; // 标记为白色
                        growPtVector.push_back(pToGrowing);                    // 将下一个生长点压入栈中
                    }
                }
            }
        }
    }
    return growImage.clone();
}

// 得到点云图像坐标系下的模板
void getPointCloudImage(pcl::PointCloud<pcl::PointXYZ>::Ptr cloud, cv::Mat &mask, const CamIntrinsic &intrinsic)
{
    float fx = intrinsic.fx;
    float fy = intrinsic.fy;
    float cx = intrinsic.cx;
    float cy = intrinsic.cy;
    int imageWidth = mask.cols;
    int imageHeight = mask.rows;
    int size = cloud->size();
    for (int i = 0; i < size; i++)
    {
        float ptx = (*cloud)[i].x;
        float pty = (*cloud)[i].y;
        float ptz = (*cloud)[i].z;
        // 归一化坐标系下的点
        float unitX = ptx / ptz;
        float unitY = pty / ptz;
        float unitZ = 1;
        float currentU = fx * unitX + cx;
        float currentV = fy * unitY + cy;
        int u = floor(currentU + 0.5);
        int v = floor(currentV + 0.5);
        // 框里面
        if (u < 0 || u >= imageWidth || v < 0 || v >= imageHeight)
        {
            continue;
        }
        // 将当前像素设置为255
        mask.at<uchar>(v, u) = 255;
    }
    return;
}


#pragma region AlgCommonRegion
void getBOXformCloud2(const pcl::PointCloud<pcl::PointXYZ> &cloudin, ObjectOBBbox &obb, ObjectAABBbox &aabb)
{
    pcl::MomentOfInertiaEstimation<pcl::PointXYZ> feature_extractor;
    feature_extractor.setInputCloud(cloudin.makeShared());
    feature_extractor.compute();
    pcl::PointXYZ minaabb;
    pcl::PointXYZ maxaabb;
    pcl::PointXYZ minobb;
    pcl::PointXYZ maxobb;
    feature_extractor.getAABB(minaabb, maxaabb);
    feature_extractor.getOBB(minobb, maxobb, obb.translation, obb.rotational_matrix);
    pcl::copyPoint(minaabb, aabb.min_point);
    pcl::copyPoint(maxaabb, aabb.max_point);
    pcl::copyPoint(minobb, obb.min_point);
    pcl::copyPoint(maxobb, obb.max_point);
    obb.width = obb.max_point.x - obb.min_point.x;
    obb.height = obb.max_point.y - obb.min_point.y;
    obb.depth = obb.max_point.z - obb.min_point.z;
}


// Eigen::Matrix4f CreateRotateMatrix(Eigen::Vector3f before, Eigen::Vector3f after)
// {
//     before.normalize();
//     after.normalize();

//     float angle = acos(before.dot(after));
//     Eigen::Vector3f p_rotate = before.cross(after);
//     p_rotate.normalize();

//     Eigen::Matrix4f rotationMatrix = Eigen::Matrix4f::Identity();
//     rotationMatrix(0, 0) = cos(angle) + p_rotate[0] * p_rotate[0] * (1 - cos(angle));
//     rotationMatrix(0, 1) = p_rotate[0] * p_rotate[1] * (1 - cos(angle) - p_rotate[2] * sin(angle));
//     return rotationMatrix;
// }

Eigen::Matrix4f CreateRotateMatrix(Eigen::Vector3f before, Eigen::Vector3f after)
{
    Eigen::Matrix3d rotMatrix;
    Eigen::Vector3d before_d = before.cast<double>();
    Eigen::Vector3d after_d = after.cast<double>();
    
    rotMatrix = Eigen::Quaterniond::FromTwoVectors(before_d, after_d).toRotationMatrix();
    Eigen::Matrix4f rotationMatrix = Eigen::Matrix4f::Identity();
    rotationMatrix.block<3,3>(0,0) = rotMatrix.cast<float>();
    return rotationMatrix;
}
Eigen::Matrix4f getCloudToSpecialVectorR(const pcl::PointCloud<pcl::PointXYZ> &cloud,Eigen::Vector3f after, int maxIteration, float distanThre,PlaneMode& planemodel)
{
    Eigen::Matrix4f T = Eigen::Matrix4f::Identity();
    // plane
    pcl::PointIndices::Ptr inliers{new pcl::PointIndices};
    pcl::ModelCoefficients::Ptr coefficientsMove(new pcl::ModelCoefficients());
    pcl::PointCloud<pcl::PointXYZ>::Ptr cloudMovePlaneInliers(new pcl::PointCloud<pcl::PointXYZ>);

    // Create the segmentation object
    pcl::SACSegmentation<pcl::PointXYZ> seg;
    // Optional
    seg.setOptimizeCoefficients(true);
    // Mandatory
    seg.setModelType(pcl::SACMODEL_PLANE);
    seg.setMethodType(pcl::SAC_RANSAC);
    seg.setMaxIterations(maxIteration);
    seg.setDistanceThreshold(distanThre);
    // Segment the largest planar component from the remaining cloud
    seg.setInputCloud(cloud.makeShared());
    seg.segment(*inliers, *coefficientsMove);
    // std::cout<<"输入点云点数量："<<cloud.size()<< std::endl;
    // std::cout<<"得到的内点数量："<<inliers->indices.size()<<std::endl;

    float a_move = coefficientsMove->values[0];
    float b_move = coefficientsMove->values[1];
    float c_move = coefficientsMove->values[2];
    float d_move = coefficientsMove->values[3];
    Eigen::Vector3f before = Eigen::Vector3f(a_move, b_move, c_move);
    double cosValNew=before.dot(after) /(before.norm()*after.norm());
    
    if(cosValNew<0)
    {
        a_move = -a_move;
        b_move = -b_move;
        c_move = -c_move;
        d_move = -d_move;
    }
    before = Eigen::Vector3f(a_move, b_move, c_move);
    
    Eigen::Matrix4f rotateT = CreateRotateMatrix(before, after);
    planemodel.a = a_move; 
    planemodel.b = b_move; 
    planemodel.c = c_move; 
    planemodel.d = d_move; 
    return T;
}

Eigen::Matrix4f getCloudToFloorT(const pcl::PointCloud<pcl::PointXYZ> &cloud, float distanThre,PlaneMode& planemodel)
{
    Eigen::Matrix4f T = Eigen::Matrix4f::Identity();
    // plane
    pcl::PointIndices::Ptr inliers{new pcl::PointIndices};
    pcl::ModelCoefficients::Ptr coefficientsMove(new pcl::ModelCoefficients());
    pcl::PointCloud<pcl::PointXYZ>::Ptr cloudMovePlaneInliers(new pcl::PointCloud<pcl::PointXYZ>);

    // Create the segmentation object
    pcl::SACSegmentation<pcl::PointXYZ> seg;
    // Optional
    seg.setOptimizeCoefficients(true);
    // Mandatory
    seg.setModelType(pcl::SACMODEL_PLANE);
    seg.setMethodType(pcl::SAC_RANSAC);
    seg.setMaxIterations(50);
    seg.setDistanceThreshold(distanThre);
    // Segment the largest planar component from the remaining cloud
    seg.setInputCloud(cloud.makeShared());
    seg.segment(*inliers, *coefficientsMove);
    // std::cout<<"输入点云点数量："<<cloud.size()<< std::endl;
    // std::cout<<"得到的内点数量："<<inliers->indices.size()<<std::endl;

    float a_move = coefficientsMove->values[0];
    float b_move = coefficientsMove->values[1];
    float c_move = coefficientsMove->values[2];
    float d_move = coefficientsMove->values[3];
    if(c_move<0)
    {
        a_move = -a_move;
        b_move = -b_move;
        c_move = -c_move;
        d_move = -d_move;
    }

    // 提取内点
    // pcl::PointCloud<pcl::PointXYZ>::Ptr cloudInliers(new pcl::PointCloud<pcl::PointXYZ>);
    // pcl::copyPointCloud(cloud, *inliers, *cloudInliers);
    Eigen::Vector3f before = Eigen::Vector3f(a_move, b_move, c_move);
    Eigen::Vector3f after = Eigen::Vector3f(0, 0, 1);
    Eigen::Matrix4f rotateT = CreateRotateMatrix(before, after);

    Eigen::Vector4f centroid;
    // pcl::compute3DCentroid(cloud,*inliers, centroid);
    pcl::PointCloud<pcl::PointXYZ> cloudAfterRotated;
    pcl::transformPointCloud(cloud,*inliers,cloudAfterRotated,rotateT);
    pcl::compute3DCentroid(cloudAfterRotated, centroid);

    // Eigen::Matrix4f t = Eigen::Matrix4f::Identity();
    T(2, 3) = -centroid.z();

    planemodel.a = a_move; 
    planemodel.b = b_move; 
    planemodel.c = c_move; 
    planemodel.d = d_move; 
    return T;
}

//  void getFiles(const std::string& path, std::vector<std::string>& files)
//  {
//  	files.clear();
//  	//文件句柄
//  	long long hFile = 0;
//  	//文件信息，_finddata_t需要io.h头文件
//  	struct _finddata_t fileinfo;
//  	std::string p;
//  	int i = 0;
//  	if ((hFile = _findfirst(p.assign(path).append("\\*").c_str(), &fileinfo)) != -1)
//  	{
//  		do
//  		{
//  			//如果是目录,迭代之
//  			//如果不是,加入列表
//  			if ((fileinfo.attrib & _A_SUBDIR))
//  			{
//  				//if (strcmp(fileinfo.name, ".") != 0 && strcmp(fileinfo.name, "..") != 0)
//  				//getFiles(p.assign(path).append("\\").append(fileinfo.name), files);
//  			}
//  			else
//  			{
//  				files.push_back(p.assign(path).append("\\").append(fileinfo.name));
//  			}
//  		} while (_findnext(hFile, &fileinfo) == 0);
//  		_findclose(hFile);
//  	}
//  }


//基本函数
/**
 * @brief Fit polynomial using Least Square Method.
 * 
 * @param X X-axis coordinate vector of sample data.
 * @param Y Y-axis coordinate vector of sample data.
 * @param orders Fitting order which should be larger than zero. 
 * @return Eigen::VectorXf Coefficients vector of fitted polynomial.
 */
bool FitterLeastSquareMethod(std::vector<float> &X, std::vector<float> &Y, int startIndex, int endIndex,uint8_t orders, Eigen::VectorXd& ret)
{
    // // abnormal input verification
    // if (X.size() < 2 || Y.size() < 2 || X.size() != Y.size() || orders < 1)
    //     return false;
    // if(endIndex>X.size()-1||endIndex<startIndex||startIndex<0)
    //     return false;
    // // map sample data from STL vector to eigen vector
    // Eigen::Map<Eigen::VectorXf> sampleX(X.data()+ startIndex, endIndex - startIndex + 1);
    // Eigen::Map<Eigen::VectorXf> sampleY(Y.data()+ startIndex, endIndex - startIndex + 1);

    // int sizeNUM = endIndex - startIndex + 1;
    // Eigen::MatrixXf mtxVandermonde(sizeNUM, orders + 1);  // Vandermonde matrix of X-axis coordinate vector of sample data
    // Eigen::VectorXf colVandermonde = sampleX;              // Vandermonde column

    // // construct Vandermonde matrix column by column
    // for (size_t i = 0; i < orders + 1; ++i)
    // {
    //     if (0 == i)
    //     {
    //         mtxVandermonde.col(0) = Eigen::VectorXf::Constant(sizeNUM, 1, 1);
    //         continue;
    //     }
    //     if (1 == i)
    //     {
    //         mtxVandermonde.col(1) = colVandermonde;
    //         continue;
    //     }
    //     colVandermonde = colVandermonde.array()*sampleX.array();
    //     mtxVandermonde.col(i) = colVandermonde;
    // }

    // // calculate coefficients vector of fitted polynomial
    // ret = (mtxVandermonde.transpose()*mtxVandermonde).inverse()*(mtxVandermonde.transpose())*sampleY;

    return true;
}

bool fittingLine2DSVD(std::vector<float> &X, std::vector<float> &Y,int startIndex,int endIndex,line2DTools& line)
{
    if (X.size() < 2 || Y.size() < 2 || X.size() != Y.size())
        return false;
    if(endIndex>X.size()-1||endIndex<startIndex||startIndex<0)
        return false;
        // map sample data from STL vector to eigen vector
    Eigen::Map<Eigen::VectorXf> sampleX(X.data()+ startIndex, endIndex - startIndex + 1);
    Eigen::Map<Eigen::VectorXf> sampleY(Y.data()+ startIndex, endIndex - startIndex + 1);
    Eigen::VectorXf ret = sampleX.bdcSvd(Eigen::ComputeThinU | Eigen::ComputeThinV).solve(sampleY);
    float theta0 = ret[0];
    float theta1 = ret[1];
    line.m_a = theta0;
    line.m_b = -1;
    line.m_c = theta1;
    // line.getk();
}
bool fittingLine2D(std::vector<float> &X, std::vector<float> &Y, int startIndex, int endIndex,line2DTools& line)
{
    Eigen::VectorXd ret;
    bool fitflag = FitterLeastSquareMethod(X,Y,startIndex,endIndex,1,ret);
    if (fitflag)
    {
        //theta0 *x +theta1 = y
        float theta0 = ret[0];
        float theta1 = ret[1];
        line.m_a = theta0;
        line.m_b = -1;
        line.m_c = theta1;
        // line.getk();
        return fitflag;
    }
    else
        return false;
    
}
