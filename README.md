# PayMaya-Old-Documentation
https://s3-us-west-2.amazonaws.com/developers.paymaya.com.pg/checkout/checkout.html


<?php

namespace App;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Mail;
use App\Mail\CheckoutMail;

class Transaction extends Model
{
    public static function checkout($param){
        $property = DB::table('properties')
            ->select('id', 'guest_count', 'title', 'host_id')
            ->where('id', $param->property_id)
            ->get();

        if ($property[0]->guest_count < $param->property_guest_count) {
            return ['code' => 201, 'message' => 'failed', 'data' => 'guest_count must be lessthan or equal to '.$property[0]->guest_count];
        }// Guest count should be lessthan or equal to property guest count


        //**********************************************************************************************
        // -> GENERAT TRANSACTION NUMBER
        //**********************************************************************************************
        if (isset($param->mobile)) {
            $transaction_number = self::generate_transaction_number_for_mobile($property[0]->id);
        }else{
            $transaction_number = self::generate_transaction_number($property[0]->id);
        }

        //**********************************************************************************************
        // -> SEND DETAILS TO PAYMAYA
        //**********************************************************************************************
 
        $check_out_pass_to_api = [
            "items" => [
                [
                    "name" => $property[0]->title,
                    "quantity" => $param->quantity,
                    "totalAmount" => [
                        "value" => $param->total_amount
                    ]
                ]// array of item
            ],
            "buyer" => [
                "firstName" => $param->first_name,
                "lastName" => $param->last_name,
                "contact" => [
                  "phone" => $param->contact_number,
                  "email" => $param->email
                ],
              ],
            "totalAmount" => [
                "currency" => "PHP",
                "value" => $param->total_amount,
                "details" => [
                    "discount" => "0",
                    "serviceCharge" => $param->service_fee + $param->payment_gateway_fee + $param->cleaning_fee,
                    "shippingFee" => "0",
                    "tax" => $param->vat,
                    "subtotal" => $param->subtotal
                ]
            ],
            "requestReferenceNumber" => $transaction_number,
            "redirectUrl"=> [
                "success"=> url('/transaction/checkout/status/'.$transaction_number),
                "failure"=> url('/transaction/checkout/failure/'.$transaction_number),
                "cancel"=> url('/transaction/checkout/cancel/'.$transaction_number),
            ]
        ];

        $ch = curl_init(config('app.PAYMAYA_END_POINT'));
        curl_setopt($ch, CURLOPT_POST, true);
        curl_setopt($ch, CURLOPT_HTTPHEADER, array("Accept: application/json","Authorization: Basic ".base64_encode(config('app.PAYMAYA_PUBLIC_KEY'))));
        curl_setopt($ch, CURLOPT_POSTFIELDS, urldecode(http_build_query($check_out_pass_to_api)));//->  urldecode(http_build_query($data)) allows curl request to pass a mumultidimensional array

        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        $res = curl_exec($ch);
        curl_close($ch);
        //**********************************************************************************************
        // -> SAVE TRANSACTION INFORMATION TO DATABASE
        //**********************************************************************************************
        $check_out_details = [
            'property_id' => $property[0]->id,
            'guest_id' => $param->guest_id,
            'payment_method' => 1,
            'transaction_number' => $transaction_number,
            'checkout_id' => json_decode($res)->checkoutId,
            'host_id' => $property[0]->host_id,
            'first_name' => $param->first_name,
            'last_name' => $param->last_name,
            'email' => $param->email,
            'contact_number' => $param->contact_number,
            'message_to_host' => $param->message_to_host,
            'guest_count' => $param->guest_count,
            'booking_start_date' => $param->start_date,
            'booking_end_date' => $param->end_date,
            'price' => $param->price,
            'quantity' => $param->quantity,
            'subtotal' => $param->subtotal,
            'vat' => $param->vat,
            'payment_gateway_fee' => $param->payment_gateway_fee,
            'service_fee' => $param->service_fee,
            'cleaning_fee' => $param->cleaning_fee,
            'total_amount' => $param->total_amount,
            'payment_condition' => 'Abandoned',
            'payment_status' => 3,
            'status' => 0,
            'created_at' => now(),
            'updated_at' => now()
        ];

        $insert_transaction = DB::table('transactions')->insertGetId($check_out_details);

        return json_encode([
            'code' => 200,
            'message' => 'success',
            'data' => $res,
            'payment_status_urls' => [
                'success_url' => url('/transaction/checkout/status/'.$transaction_number),
                "failure"=> url('/transaction/checkout/failure/'.$transaction_number),
                "cancel"=> url('/transaction/checkout/cancel/'.$transaction_number),
            ]
        ]);
    }

    public static function get_and_update_checkout_details($transaction_number){
        $get_transaction = DB::table('transactions as t')
            ->select(
                't.transaction_number', 't.receipt_number', 't.booking_start_date', 't.booking_end_date', 't.property_id', 't.guest_count',
                't.price', 't.quantity', 't.subtotal', 'vat', 'payment_gateway_fee', 't.service_fee', 't.cleaning_fee', 't.total_amount', 't.created_at',
                't.checkout_id', 't.email as user_email', 'p.title', 'p_add.province_text', 'p_add.city_municipality_text', 'p_add.barangay_text', 'p_add.complete_address',
                't.first_name', 't.last_name', 'h.host_name', 'h.email as host_email', 'h.mobile', 'u.first_name as user_first_name',
                'u.last_name as user_last_name', 'u.mobile_number as user_mobile_number',
                'pt.name as property_type'
            )
            ->join('properties as p', 'p.id', '=', 't.property_id')
            ->join('property_types as pt', 'pt.id', '=', 'p.type')
            ->join('property_addresses as p_add', 'p_add.property_id', '=', 'p.id')
            ->join('users as u', 'u.id', '=', 't.guest_id')
            ->join('hosts as h', 'h.host_id', '=', 't.host_id')
            ->where('t.transaction_number', $transaction_number)
            ->get();

        if (count($get_transaction) < 1) {
            return json_encode(['code' => 201, 'message' => 'failed']);
        }

        $curl = curl_init();
        curl_setopt_array($curl, array(
            CURLOPT_URL => config('app.PAYMAYA_END_POINT').'/'.$get_transaction[0]->checkout_id,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_CUSTOMREQUEST => "GET",
            CURLOPT_HTTPHEADER => array(
                "Authorization: Basic ".base64_encode(config('app.PAYMAYA_SECRET_KEY')),
                "Content-Type: application/json",
            ),
        ));

        $response = curl_exec($curl);
        $err = curl_error($curl);
        curl_close($curl);

        $payment_status = 2; // Failed
        if (json_decode($response)->paymentStatus == 'PAYMENT_SUCCESS') {
            $payment_status = 1; // Success
        }
        
        DB::table('transactions')->where('checkout_id', $get_transaction[0]->checkout_id)
        ->update([
            'receipt_number' => json_decode($response)->receiptNumber,
            'payment_condition' => json_decode($response)->status,
            'payment_type' => json_decode($response)->paymentScheme,
            'payment_status' => $payment_status,
            'status' => $payment_status, // status is also same values of payment 
            'payment_at' => date("Y-m-d H:i:s", strtotime(json_decode($response)->createdAt)),
            'updated_at' => now()
        ]);

        /*|------------------------------EMAILS----------------------------------|*/
        $transaction = $get_transaction[0];

        $transaction->for = 'Booker';
        Mail::to([$get_transaction[0]->user_email, 'helpdesk@becase.ph'])->send(new CheckoutMail($transaction)); // Email for Booker

        $transaction->for = 'Host';

        Mail::to([$get_transaction[0]->host_email, 'helpdesk@becase.ph'])->send(new CheckoutMail($transaction)); // Email for Host

        return json_encode(['code' => 200, 'message' => 'success', 'data' => $get_transaction]);
    }
    
    public static function generate_transaction_number($property_id){
        $transaction_count = DB::select(DB::raw("SELECT COUNT(*) as num FROM transactions"));
        // BE(listing_id)(YEAR)(MONTH)(DAY)(Transaction_count)
        return 'BE'.$property_id.date('Ymd') .$transaction_count[0]->num.strtoupper(token_generator(13));
    }

    public static function generate_transaction_number_for_mobile($property_id){
        $transaction_count = DB::select(DB::raw("SELECT COUNT(*) as num FROM transactions"));
        // BE(listing_id)(YEAR)(MONTH)(DAY)(Transaction_count)
        return 'BEM'.$property_id.date('Ymd') .$transaction_count[0]->num.strtoupper(token_generator(13));
    }
}




