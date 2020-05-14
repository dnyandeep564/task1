//# task1
//it is a pkg that subscribes to image and publishes coordinates of the cntre of green box
   #include <ros/ros.h>
   #include <image_transport/image_transport.h>
   #include <cv_bridge/cv_bridge.h>
   #include <sensor_msgs/image_encodings.h>
   #include <opencv2/imgproc/imgproc.hpp>                          //inc
   #include <opencv2/highgui/highgui.hpp>
   #include "opencv2/imgproc.hpp"
   #include "opencv2/imgcodecs.hpp"
   #include "opencv2/highgui.hpp"
   #include <iostream>
   
   #include <sstream>
    
   static const std::string OPENCV_WINDOW = "Image window";
    
   class ImageConverter
   {
     ros::NodeHandle nh_;
     image_transport::ImageTransport it_;                                     //rosnode
     image_transport::Subscriber image_sub_;
     image_transport::Publisher image_pub_;
     
     ros::Publisher chatter_pub = n.advertise<std_msgs::String>("chatter", 100);    //pub 
     ros::Rate loop_rate(30);
        
   public:
     ImageConverter()
       : it_(nh_)
     {
       // Subscrive to input video feed and publish output video feed
       image_sub_ = it_.subscribe("/magnus/camera/image_raw", 1,
         &ImageConverter::imageCb, this);
       image_pub_ = it_.advertise("/image_converter/output_video", 1);                 //subscribed to image
   
       cv::namedWindow(OPENCV_WINDOW);
     }
   
     ~ImageConverter()
     {
       cv::destroyWindow(OPENCV_WINDOW);
     }
   
     void imageCb(const sensor_msgs::ImageConstPtr& msg)
     {
       cv_bridge::CvImagePtr cv_ptr;
       try
       {
         cv_ptr = cv_bridge::toCvCopy(msg, sensor_msgs::image_encodings::BGR8);
       }
       catch (cv_bridge::Exception& e)
       {
         ROS_ERROR("cv_bridge exception: %s", e.what());
         return;
       }
   
       mat dst,src_gray;
        cvtColor( cv_ptr->image, src_gray, COLOR_BGR2GRAY ); // Convert the image to Gray
       
       double thresh = 50;

       double maxValue = 255;
       
         // Binary Threshold

       threshold(src_gray,dst, thresh, maxValue, THRESH_BINARY);
       cv_ptr->image = dst;

       // Update GUI Window
       cv::imshow(OPENCV_WINDOW, cv_ptr->image);
       cv::waitKey(3);
   
       // Output modified video stream
       image_pub_.publish(cv_ptr->toImageMsg());
       
       Mat canny_output;
       Canny( cv_ptr->image, canny_output, thresh, thresh*2 );
       vector<vector<Point> > contours;
       findContours( canny_output, contours, CV_RETR_EXTERNAL, CHAIN_APPROX_SIMPLE );
       
       vector<vector<Point> > contours_poly( contours.size() );
       vector<Rect> boundRect( contours.size() );
       
       for( size_t i = 0; i < contours.size(); i++ )
       {
           approxPolyDP( contours[i], contours_poly[i], 3, true );
           boundRect[i] = boundingRect( contours_poly[i] );
           
           std_msgs::String msg;
   
           std::stringstream ss;
           ss << "x coordinate = " << (boundRect[i].tl.x + boundRect[i].br.x)/2 << "y coordinate = " << (boundRect[i].tl.y + boundRect[i].br.y)/2 ;
           msg.data = ss.str();
     
           ROS_INFO("%s", msg.data.c_str());
           
           chatter_pub.publish(msg);
           loop_rate.sleep();
       }
     }
   };
   
   int main(int argc, char** argv)
   {
     ros::init(argc, argv, "image_converter");
     ImageConverter ic;
     ros::spin();
     return 0;
   }
