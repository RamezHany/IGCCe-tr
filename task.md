Project Brief: IGCC Learning Management System (LMS)1. OverviewThe goal of this project is to build a complete Learning Management System (LMS) for IGCC. The platform will allow users to browse, purchase, and watch online courses. It will also provide an admin panel for managing courses, users, and content. The entire application will be built using Next.js, with Google Sheets acting as the primary database.2. Technology StackFramework: Next.js (for both Frontend and Backend/API Routes)Authentication: NextAuth.jsStyling: Tailwind CSSUI Components: shadcn/uiDatabase: Google Sheets API (via a Service Account)File Storage: Cloudflare R2 (for video files, images, and regular sheet backups)Video Processing & Security: Mux (for secure video streaming and anti-download measures)Payment Gateway: Paymob3. User FlowThis section describes the typical journey of a user on the platform.Browse & Discover: A new user lands on the homepage, which is the main courses page. They can browse and filter available courses.Enroll in a Course: When a user decides to enroll, they are prompted to create an account or log in.Account Creation: The user registers with their details (name, email, password, etc.). This data is saved to the users-sheet in Google Sheets.Purchase Course: After logging in, the user is redirected to the payment page. They will complete the purchase using the Paymob payment gateway.Course Access: Once the payment is successfully confirmed via a Paymob webhook, the user's courses-status is updated in the users-sheet, and they gain access to the course content.Learning & Progress Tracking: The user can watch course videos. The system must track their progress (e.g., "watched 10 out of 15 videos"). This progress data is stored in the courses-status JSON object for that user.Certificate of Attendance: After watching all the videos in a course, the user is automatically eligible for a free "Certificate of Attendance." They can view or download it from their profile.International Certificate: The user has the option to purchase an "International Certificate." This involves:Making another payment through our Paymob integration.Upon successful payment, our system sends an automated notification (e.g., via email or another API call) to the IGRC system with the user's details.The IGRC system then handles the exam and certificate issuance directly with the user. Our responsibility ends after a successful payment notification.Certificate Verification: All certificates will have a QR code. When scanned, it should link to a verification page on the IGCC website that confirms the certificate's authenticity.User Profile: The user can access their profile page to:View and edit their personal data.See a list of all courses they are enrolled in.Track their progress for each course.Access their certificates.4. Admin FlowThis section describes the administrative capabilities.Login: The admin logs in using credentials stored in the .env file.Course Management:Add a New Course: Admins can create a new course, adding details like name, description, price, presenter info, and a thumbnail.Upload Videos: Videos should be uploaded directly from a client-side component in the admin panel to a Cloudflare R2 bucket. Once the upload is complete, a backend process should be triggered to add the video from R2 to Mux as a new asset. The Mux player URL and asset ID are then stored in the content-json for the course.Structure Content: Admins can structure a course into parts/chapters and add video lessons to each part.Set Course Type: Label courses as "Recorded," "Online Live," etc.Add Certificates: Upload the certificate template URL for each course.User Management:View all registered users and their details.Check a user's enrollment status, course progress, and payment history (including Paymob Order ID).Analytics:View basic analytics on course enrollment and user activity.5. Developer Flow & Database Schema (Google Sheets)The application will use a single Google Sheet with multiple sub-sheets (tabs). On application startup, you must implement logic to check if the required sub-sheets exist. If not, create them and add the specified header rows.Sheet Name: users-sheetidnameemailphone-numberagegenderpasswordcourses-status (JSON)demo-videouuidstringstringstringnumberstringhashed_stringobjectmux_urlcourses-status JSON Structure:[
  {
    "course_id": "unique_course_id_1",
    "payment_status": "completed",
    "order_id": "paymob_order_id",
    "progress": {
      "watched_videos": 10,
      "total_videos": 15
    },
    "certificates": {
        "attendance_cert_url": "url_if_completed",
        "international_cert_paid": false
    }
  }
]
Sheet Name: courses-sheetidnamecourse-presenter (JSON)thumbnail-urlcert-urlpricetype (JSON)content-jsoncourse-descriptionuuidstringobjectr2_public_urlr2_public_urlnumberobjectobjectstringcourse-presenter JSON Structure:{
  "name": "Presenter Name",
  "img_url": "url_to_presenter_image_on_r2"
}
type JSON Structure:{
  "category": "online",
  "details": "recorded" 
}
// Or for live sessions
{
  "category": "live",
  "details": "zoom",
  "registration_end": "ISO_date",
  "max_attendees": 100
}
content-json Structure:[
  {
    "part_name": "Introduction",
    "videos": [
      {
        "video_name": "Welcome to the Course",
        "mux_asset_id": "mux_id",
        "mux_playback_id": "mux_playback_id"
      }
    ]
  }
]
Sheet Name: course-resources-sheetThis sheet stores the direct links to the source files as a backup.course-idlinks-jsonuuidobjectlinks-json Structure:{
  "videos": [
    {
      "name": "video_name.mp4",
      "r2_url": "cloudflare_r2_url"
    }
  ],
  "images": [
     {
      "name": "thumbnail.png",
      "r2_url": "cloudflare_r2_url"
    }
  ]
}
6. Asset & Video Management FlowImages: All images (thumbnails, presenter photos, etc.) must be uploaded to a dedicated /images folder in the Cloudflare R2 bucket.Videos: All raw video files must be uploaded from the client-side admin panel to a /videos folder in the Cloudflare R2 bucket.Mux Integration: After a video successfully uploads to R2, trigger a server-side function. This function will tell Mux to ingest the video directly from the R2 URL.Database Update: Once Mux creates the asset and provides a playback ID, update the content-json in the courses-sheet with the Mux details.7. Frontend & UI/UXFrameworks: Use Tailwind CSS for styling and shadcn/ui for pre-built, accessible components.Color Palette:Primary Green: #65b351Dark Green (for accents): #359747Dark Gray (for text): #4a4a4aLogo: The logo is named igcc-logo.png and should be placed in the /public folder.Pages to Create:Home Page (/): This is the main course listing page. It should feature filtering options. Important: To optimize load times, do not fetch the content-json column for courses on this page. Fetch only the essential data (name, price, thumbnail, etc.).Login Page (/login)Register Page (/register)Profile Page (/profile)Course Detail Page (/course/[id]): This page shows course details, a demo video, and the "Buy Now" button.8. Authentication (NextAuth.js)User Authentication: Use the CredentialsProvider. The authorize function will query the users-sheet to verify the user's email and password.Admin Authentication: Use a separate CredentialsProvider for the admin route (e.g., /admin/login). The credentials for the admin should be hardcoded in the .env file for simplicity and security.9. Paymob Integration (CRITICAL)The payment flow is crucial. It involves a webhook for server-to-server confirmation and a redirect for the user's browser. Please follow this implementation closely.A. Webhook HandlerCreate a webhook endpoint to receive notifications from Paymob when a transaction is processed.File Path: app/api/webhooks/payments/route.tsimport { NextResponse } from 'next/server';
// You will need to create these helper functions based on the Google Sheets logic
// import { updatePaymentStatus } from '@/lib/google-sheets';
// import { sendTelegramNotification } from '@/lib/telegram'; // Optional: if you want notifications

export async function POST(request: Request) {
  try {
    let data;
    try {
      data = await request.json();
      console.log('Payment webhook received:', JSON.stringify(data, null, 2));
    } catch (parseError) {
      console.error('Failed to parse webhook JSON data:', parseError);
      // Fallback for form-data or other formats if needed
      return NextResponse.json({ error: 'Invalid JSON payload' }, { status: 400 });
    }
    
    // According to Paymob docs, the 'obj.success' field indicates the transaction status.
    const isSuccess = data?.obj?.success;
    const orderId = data?.obj?.order?.id?.toString();

    if (!orderId) {
      console.error('Webhook Error: Order ID is missing from the payload.');
      return NextResponse.json({ error: 'No order ID in webhook data' }, { status: 400 });
    }

    if (isSuccess === true) {
      console.log(`Processing successful payment for order ID: ${orderId}`);
      
      // 1. Update Payment Status in Google Sheets
      // This function needs to find the user with the matching orderId in their 'courses-status' and update 'payment_status' to 'completed'.
      // const result = await updatePaymentStatusInSheet(orderId, 'completed');

      // if (!result.success) {
      //   console.error('Failed to update payment status in Google Sheets:', result.message);
      //   return NextResponse.json({ error: result.message }, { status: 500 });
      // }
      
      console.log(`Successfully updated payment status for order ID: ${orderId}`);
      
      // Optional: 2. Send a notification (e.g., Telegram, email)
      // const notificationMessage = `✅ New course payment completed!\nUser: ${result.userName}\nCourse: ${result.courseName}\nOrder ID: ${orderId}`;
      // await sendTelegramNotification(notificationMessage);

      return NextResponse.json({ message: 'Webhook processed successfully' });

    } else {
      console.log(`Payment not successful for order ID: ${orderId}, status: ${isSuccess}`);
      // Optional: Update status to 'failed' in Google Sheets
      // await updatePaymentStatusInSheet(orderId, 'failed');
      return NextResponse.json({ message: 'Payment not successful, no action taken' });
    }

  } catch (error) {
    console.error('Error processing payment webhook:', error);
    return NextResponse.json(
      { error: 'An internal error occurred' },
      { status: 500 }
    );
  }
}

// GET endpoint for simple verification that the route is up.
export async function GET() {
  return NextResponse.json({ message: 'Payment webhook endpoint is active' });
}
B. Payment Redirect HandlerAfter the user interacts with the Paymob iframe, they are redirected back to your site. This page catches the redirect, reads the URL parameters, and forwards the user to a clean success/failure page.File Path: app/payment/redirect/page.tsx'use client';

import { useEffect, useState, Suspense } from 'react';
import { useSearchParams, useRouter } from 'next/navigation';
import { Loader2 } from 'lucide-react';

function RedirectContent() {
  const router = useRouter();
  const searchParams = useSearchParams();
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const handleRedirect = () => {
      try {
        const params = Object.fromEntries(searchParams.entries());
        console.log('PayMob redirect handler received params:', params);
        
        const orderId = params.order;
        const success = params.success === 'true';
        
        if (!orderId) {
          setError('No order ID found in redirect parameters. Please contact support.');
          return;
        }

        if (success) {
          // Redirect to a clean success URL
          const redirectUrl = `/payment/success?order=${orderId}`;
          console.log(`Redirecting to success page: ${redirectUrl}`);
          router.replace(redirectUrl);
        } else {
          // Redirect to a clean failure URL
          const redirectUrl = `/payment/failure?order=${orderId}`;
          console.log(`Redirecting to failure page: ${redirectUrl}`);
          router.replace(redirectUrl);
        }
      } catch (e) {
        console.error('Error during redirection:', e);
        setError('An unexpected error occurred during redirection.');
      }
    };

    handleRedirect();
  }, [searchParams, router]);

  return (
    <div className="min-h-screen flex flex-col items-center justify-center bg-gray-50">
      {error ? (
        <div className="text-center p-6 bg-red-100 text-red-700 rounded-lg">
            <h1 className="text-xl font-bold mb-2">Redirect Error</h1>
            <p>{error}</p>
        </div>
      ) : (
        <div className="text-center">
          <Loader2 className="h-12 w-12 animate-spin text-primary mx-auto mb-4" />
          <h1 className="text-xl font-bold mb-2">Processing Payment</h1>
          <p className="text-gray-600">Please wait while we confirm your payment status...</p>
        </div>
      )}
    </div>
  );
}

export default function PayMobRedirectHandler() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <RedirectContent />
    </Suspense>
  );
}
C. Payment Success PageThis is the final page the user sees after a successful payment. It gives them clear confirmation. The actual data update should happen via the webhook, not here. This page is for UI purposes only.File Path: app/payment/success/page.tsx'use client';

import { Suspense } from 'react';
import { useSearchParams } from 'next/navigation';
import Link from 'next/link';
import { CheckCircle, ArrowLeft } from 'lucide-react';

function SuccessContent() {
    const searchParams = useSearchParams();
    const orderId = searchParams.get('order');

    return (
        <div className="min-h-screen flex items-center justify-center bg-gray-50">
            <div className="bg-white rounded-lg shadow-xl max-w-md w-full p-8 text-center">
                <CheckCircle className="mx-auto h-16 w-16 text-green-500 mb-4" />
                <h1 className="text-2xl font-bold mb-2">Payment Successful!</h1>
                <p className="text-gray-600 mb-6">
                    Your course has been unlocked. You can now access it from your profile.
                </p>
                {orderId && (
                    <div className="text-sm text-gray-500 mb-6">
                        Order ID: <span className="font-mono bg-gray-100 p-1 rounded">{orderId}</span>
                    </div>
                )}
                <Link
                    href="/profile"
                    className="w-full inline-flex items-center justify-center px-4 py-2 bg-primary text-white rounded-md hover:bg-primary/90 transition"
                >
                    <ArrowLeft className="h-4 w-4 mr-2" />
                    Go to My Courses
                </Link>
            </div>
        </div>
    );
}


export default function PaymentSuccessPage() {
  return (
    <Suspense fallback={<div>Loading confirmation...</div>}>
      <SuccessContent />
    </Suspense>
  );
}
10. Environment VariablesCreate a .env.local file with the following variables.# Google Service Account API
GOOGLE_SERVICE_ACCOUNT_EMAIL=
GOOGLE_PRIVATE_KEY=
GOOGLE_SHEET_ID=

# Cloudflare R2 Configuration
NEXT_PUBLIC_R2_ACCESS_KEY_ID=
NEXT_PUBLIC_R2_SECRET_ACCESS_KEY=
NEXT_PUBLIC_R2_BUCKET_NAME=
NEXT_PUBLIC_R2_ENDPOINT= # e.g., https://<account_id>.r2.cloudflarestorage.com
NEXT_PUBLIC_R2_REGION=auto
NEXT_PUBLIC_R2_PUBLIC_LINK= # The public URL for your R2 bucket

# Mux Configuration
MUX_TOKEN_ID=
MUX_TOKEN_SECRET=
 
# Paymob Configuration
PAYMOB_API_KEY=
PAYMOB_INTEGRATION_ID=
PAYMOB_IFRAME_ID=

# NextAuth
NEXTAUTH_SECRET= # Generate a strong secret: openssl rand -base64 32
NEXTAUTH_URL=http://localhost:3000

# Admin Credentials 
ADMIN_USERNAME=admin
ADMIN_PASSWORD=admin
11. Final Task: DocumentationAfter you have completed the development of the application, please create a documentation page available at the /docs route. This documentation should be written in English and include:Setup Guide: How to get the project running locally (cloning, installing dependencies, setting up the .env.local file).Usage Guide: A brief overview of how to use the admin panel and the main features of the platform.Good luck!