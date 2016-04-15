---
layout: post
title: "为Volley添加Https支持"
date: 2016-03-07 08:54:26 +0800
comments: true
categories: android
---

我们都知道，Volley是封装良好的Http请求框架，那它能否用于Https请求呢？  
答案是肯定的，有码为证，首先我们看一下创建请求队列的代码<!--more-->：  

    public static RequestQueue newRequestQueue(Context context,HttpStack stack)
    {
        File cacheDir=new File(context.getCacheDir(),DEFAULT_CACHE_DIR);

        String userAgent="volley/0";

        try{
            String packageName=context.getPackageName();
            PackageInfo info=context.getPackageManager().getPackageInfo(packageName,0);
            userAgent=packageName+"/"+info.versionCode;
        }catch(PackageManager.NameNotFoundException e)
        {
            e.printStackTrace();
        }

        if(stack==null)
        {
            if(Build.VERSION.SDK_INT>=9)
            {
                stack=new HurlStack();
            }else{
                stack=new HttpClientStack(AndroidHttpClient.newInstance(userAgent));
            }
        }

        Network network=new BasicNetwork(stack);

        RequestQueue queue=new RequestQueue(new DiskBasedCache(cacheDir),network);
        queue.start();

        return queue;
    }


由于真正执行请求的是Stack,我们看一下HurlStack这个类,相关代码如下：  

	 public HurlStack(){
        this(null);
    }

    public HurlStack(UrlRewriter urlRewriter)
    {
        this(urlRewriter,null);
    }

    public HurlStack(UrlRewriter urlRewriter,SSLSocketFactory sslSocketFactory)
    {
        mUrlRewriter=urlRewriter;
        mSslSocketFactory=sslSocketFactory;
    }

    @Override
    public HttpResponse performRequest(Request<?> request,Map<String,String> additionalHeaders)
            throws IOException,AuthFailureError
    {
        String url=request.getUrl();
        HashMap<String,String>map=new HashMap<>();
        map.putAll(request.getHeaders());
        map.putAll(additionalHeaders);
        if(mUrlRewriter!=null)
        {
            String rewritten=mUrlRewriter.rewriteUrl(url);
            if(rewritten==null)
            {
                throw new IOException("URL blocked by rewriter:"+url);
            }
            url=rewritten;
        }
        URL parsedUrl=new URL(url);
        HttpURLConnection connection=openConnection(parsedUrl,request);
        for(String headerName:map.keySet())
        {
            connection.addRequestProperty(headerName,map.get(headerName));
        }
        setConnectionParametersForRequest(connection,request);

        ProtocolVersion protocolVersion=new ProtocolVersion("HTTP",1,1);
        int responseCode=connection.getResponseCode();
        if(responseCode==-1)
        {
            throw new IOException("Could not retrieve response code from HttpUrlConnection.");
        }
        StatusLine responseStatus=new BasicStatusLine(protocolVersion,connection.getResponseCode(),connection.getResponseMessage());
        BasicHttpResponse response=new BasicHttpResponse(responseStatus);
        response.setEntity(entityFromConnection(connection));
        for(Map.Entry<String,List<String>> header:connection.getHeaderFields().entrySet())
        {
            if(header.getKey()!=null)
            {
                Header h=new BasicHeader(header.getKey(),header.getValue().get(0));
                response.addHeader(h);
            }
        }
        return response;
    }

    private HttpURLConnection openConnection(URL url,Request<?>request) throws IOException
    {
        HttpURLConnection connection=createConnection(url);

        int timeoutMs=request.getTimeoutMs();
        connection.setConnectTimeout(timeoutMs);
        connection.setReadTimeout(timeoutMs);
        connection.setUseCaches(false);
        connection.setDoInput(true);

        if("https".equals(url.getProtocol())&&mSslSocketFactory!=null)
        {
            ((HttpsURLConnection)connection).setSSLSocketFactory(mSslSocketFactory);
        }

        return connection;
    }


显然，HurlStack是支持Https请求的，只是由于Volley开放的接口中没有带SSLSocketFactory参数，所以调用的是HurlStack的无参构造方法，而这个默认构造方法由于sslSocketFactory为null,从而导致不支持Https请求。显然，我们要添加Https请求的话也很简单，只要在Volley中添加一个新的接口即可，如下所示：  

	 public static RequestQueue createHttpsRequestQueue(Context context,HttpStack stack,SSLSocketFactory sslSocketFactory)
    {
        File cacheDir=new File(context.getCacheDir(),DEFAULT_CACHE_DIR);
        String userAgent="volley/0";

        try
        {
            String packageName=context.getPackageName();
            PackageInfo info=context.getPackageManager().getPackageInfo(packageName,0);
            userAgent=packageName+"/"+info.versionCode;
        }catch(PackageManager.NameNotFoundException ex)
        {
            ex.printStackTrace();
        }

        if(stack==null)
        {
            if(Build.VERSION.SDK_INT>=9)
            {
                stack=new HurlStack(null,sslSocketFactory);
            }else{
                stack=new HttpClientStack(AndroidHttpClient.newInstance(userAgent));
            }
        }

        Network network=new BasicNetwork(stack);

        RequestQueue queue=new RequestQueue(new DiskBasedCache(cacheDir),network);
        queue.start();

        return queue;
    }

这样虽然解决了问题，但是实际上封装不够好，更好的方法是让用户只要提供InputStream即可，代码如下所示：  

	 private SSLSocketFactory getSocketFactory(InputStream...certificates)
    {
        try
        {
            CertificateFactory certificateFactory = CertificateFactory.getInstance("X.509");
            KeyStore keyStore = KeyStore.getInstance(KeyStore.getDefaultType());
            keyStore.load(null);
            int index = 0;
            for (InputStream certificate : certificates)
            {
                String certificateAlias = Integer.toString(index++);
                keyStore.setCertificateEntry(certificateAlias, certificateFactory.generateCertificate(certificate));

                try
                {
                    if (certificate != null)
                        certificate.close();
                } catch (IOException e)
                {
                }
            }

            SSLContext sslContext = SSLContext.getInstance("TLS");

            TrustManagerFactory trustManagerFactory =
                    TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());

            trustManagerFactory.init(keyStore);
            sslContext.init(null, trustManagerFactory.getTrustManagers(), new SecureRandom());
           
            return sslContext.getSocketFactory();
            
        } catch (Exception e)
        {
            e.printStackTrace();
        }
        return null;
    }

     public static RequestQueue createHttpsRequestQueue(Context context,HttpStack stack,InputStream...certificates)
    {
        File cacheDir=new File(context.getCacheDir(),DEFAULT_CACHE_DIR);
        String userAgent="volley/0";

        try
        {
            String packageName=context.getPackageName();
            PackageInfo info=context.getPackageManager().getPackageInfo(packageName,0);
            userAgent=packageName+"/"+info.versionCode;
        }catch(PackageManager.NameNotFoundException ex)
        {
            ex.printStackTrace();
        }

        if(stack==null)
        {
            if(Build.VERSION.SDK_INT>=9)
            {
                stack=new HurlStack(null,getSocketFactory(certificates));
            }else{
                stack=new HttpClientStack(AndroidHttpClient.newInstance(userAgent));
            }
        }

        Network network=new BasicNetwork(stack);

        RequestQueue queue=new RequestQueue(new DiskBasedCache(cacheDir),network);
        queue.start();

        return queue;
    }


比如，证书文件("srca.cer")在assets下，那么只需要进行如下调用即可：  

	RequestQueue queue=createHttpsRequestQueue(this,null,ggetAssets().open("srca.cer"));




