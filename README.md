# note
<?php

class GoogleApis_Vision {
    private static $api_key = 'AIzaSyDxz6XUrbb3mhAUaIlUTV913j2WP-boAhY';

    public static function checkImageBeforeUpload($file){
        $cvurl = "https://vision.googleapis.com/v1/images:annotate?key=" . self::$api_key;
        $type1 = "SAFE_SEARCH_DETECTION";
        $type2 = "LABEL_DETECTION";
        $result = [
            'is_adult_violence' => false,
            'message' => '',
            'translated' => '',
            'error' => false
        ];

        if ($file) {
                //convert it to base64
                $fname = $file['tmp_name'];
                $data = file_get_contents($fname);
                $base64 = base64_encode($data);

                $r_json ='{
                    "requests": [
                        {
                          "image": {
                            "content":"' . $base64. '"
                          },
                          "features": [
                              {
                                "type": "' .$type1. '",
                                "maxResults": 200
                              },{
                                "type": "' .$type2. '",
                                "maxResults": 200
                              }
                          ]
                        }
                    ]
                }';

                $curl = curl_init();
                curl_setopt($curl, CURLOPT_URL, $cvurl);
                curl_setopt($curl, CURLOPT_RETURNTRANSFER, true);
                curl_setopt($curl, CURLOPT_HTTPHEADER, array("Content-type: application/json"));
                curl_setopt($curl, CURLOPT_POST, true);
                curl_setopt($curl, CURLOPT_POSTFIELDS, $r_json);
                $json_response = curl_exec($curl);
                $status = curl_getinfo($curl, CURLINFO_HTTP_CODE);
                curl_close($curl);

                if ( $status != 200 ) {
                    $result = [
                        'message' => "Error: $cvurl failed status $status",
                        'error' => true
                    ];
                }
                $json_response = json_decode($json_response);
                //get label of image
                $labelAnnotations = $json_response->responses[0]->labelAnnotations;
                $description = [];
                foreach ($labelAnnotations as $labelAnnotation) {
                    $description[] =  $labelAnnotation->description;
                }
                $translated = self::saveLabel($description);
                $result['translated'] = $translated;
                //is adult or violence
                $safeSearchAnnotation = $json_response->responses[0]->safeSearchAnnotation;
                if($safeSearchAnnotation->adult == "POSSIBLE" || $safeSearchAnnotation->violence == "POSSIBLE"){
                    $result['is_adult_violence' ]= true;
                    $result['message' ]= 'is adult or violence';
                }
            }
            else {
                $result = [
                    'message' => "Error! Input A Null File !",
                    'error' => true
                ];
            }
        return $result;
    }
    private static function saveLabel($descriptions){
        $array = [];
        $arrayTranslated = [];
        foreach ($descriptions as $key => $description){
            if(!Dictionarys::checkDictionarys($description)){
                $array['english'] =  $description;
                $array['japanese'] =  self::translateText($description);
                $arrayTranslated[] = $array['japanese'];
                Dictionarys::saveAction($array);
            }
        }
        return $arrayTranslated;
    }
    private static function translateText($text){
        $cvurl = "https://translation.googleapis.com/language/translate/v2?key=" . self::$api_key.'&target=ja'.'&q='.rawurlencode($text);
        $handle = curl_init($cvurl);
        curl_setopt($handle, CURLOPT_RETURNTRANSFER, true);
        $response = curl_exec($handle);
        $responseDecoded = json_decode($response, true);
        $responseCode = curl_getinfo($handle, CURLINFO_HTTP_CODE);      //Here we fetch the HTTP response code
        curl_close($handle);
        return $responseDecoded['data']['translations'][0]['translatedText'];
    }

}

?>
