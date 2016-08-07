---
layout: default
title: 3dApi framework (C++)
category: 3dApi
---
<div class="content-inner">
    <div class="container">
        <h1>3dApi framework</h1>

        <h3>Project:</h3>
        <p>
        	<a href="/projects/3dapi.html">3dApi</a>
        </p>

        <h3>Programming language:</h3>
        <p>
        	C++
        </p>

        <h3>Github:</h3>
        <p>
        <a href="https://github.com/gpanic/3dApi">https://github.com/gpanic/3dApi</a>
        </p>

        <h3>Description</h3>
        <p>
          3dApi uses this framework to make the creation of Direct3D and OpenGL apps easier. The class App3D uses the Win32 API to create a window, manage input, run the main game loop and optionally record the execution time of the most important functions. DXApp and GLApp handle the API specific initialization that is needed to start drawing on the window. Each application in 3dApi derives from either DXApp or GLApp and overrides the InitScene, Update and Render functions to draw different content onto the window.
      	</p>

      	<h3>App3D.h</h3>
{% highlight cpp %}
#pragma once
#include <windows.h>
#include <string>
#include <sstream>
#include <fstream>
#include <vector>
#include <iomanip>
#include "Util.h"
#include "Vertex.h"
#include "Vector.h"
#include "Material.h"
#include "Input.h"

class App3D
{
public:
    App3D(HINSTANCE hInstance);
    virtual ~App3D();

    bool benchmarking = false;
    bool processInput = true;
    bool update = true;
    bool saveDeviceInfo = false;
    int benchmarkFrameCount = 10000;
    std::string assetPath = "Assets/";
    std::string shaderPath;
    std::string binaryPath;
    std::string modelPath;
    Color bgColor;
    Input input;

    int Run();
    virtual LRESULT WndProc(HWND hWnd, UINT msg, WPARAM wParam, LPARAM lParam);

protected:
    double          mDeltaTime = 0;
    float           mFPS = 0;
    std::string     mBenchmarkResultName;
    std::string     mDeviceInfoFileName;
    HWND            mWindow;
    HINSTANCE       mInstance;
    unsigned int    mHeight;
    unsigned int    mWidth;
    std::string     mAppTitle;
    std::string     mWindowClass;
    unsigned long   mWindowStyle;

    virtual bool InitAPI() = 0;
    virtual bool InitScene() = 0;
    virtual void Update() = 0;
    virtual void Render() = 0;
    virtual void UpdateWindowTitle() = 0;
    virtual void SwapBuffer() = 0;
    virtual void SaveSnapshot(std::string file) = 0;
    virtual void SaveDeviceInfo(std::string file) = 0;

private:
    __int64         mPreviousCount = 0;
    __int64         mCurrentCount = 0;
    __int64         mFrequency = 0;
    float           mElapsedTime = 0;
    unsigned int    mDeltaFrameCount = 0;
    std::vector<float>  mFrameTimes;
    unsigned int    mFrameCount = 0;
    bool            mInLoop = false;

    float mUpdateTime = 0;
    float mRenderTime = 0;
    float mSwapBufferTime = 0;

    std::vector<float>  mUpdateTimes;
    std::vector<float>  mRenderTimes;
    std::vector<float>  mSwapBufferTimes;

    bool firstSnapshot = false;

    bool InitWindow();
    int MsgLoop();
    void StartTime();
    void UpdateDeltaTime();
    void UpdateTime();
    bool Benchmark();
};
{% endhighlight %}

      	<h3>App3D.cpp</h3>
{% highlight cpp %}
#include "App3D.h"

namespace
{
    App3D* gApp = NULL;
}

LRESULT CALLBACK MainWndProc(HWND hWnd, UINT msg, WPARAM wParam, LPARAM lParam)
{
    if (gApp)
        return gApp->WndProc(hWnd, msg, wParam, lParam);
    else
        return DefWindowProc(hWnd, msg, wParam, lParam);
}

App3D::App3D(HINSTANCE hInstance)
{
    gApp = this;

    mInstance = hInstance;
    mWindow = NULL;
    mWidth = 800;
    mHeight = 800;
    mAppTitle = "3D App";
    mWindowClass = "3DAPPWNDCLASS";
    mWindowStyle = WS_OVERLAPPEDWINDOW;
    mBenchmarkResultName = mAppTitle + " Result.txt";

    binaryPath = assetPath + "Binary/";
    modelPath = assetPath + "Models/";
}

App3D::~App3D()
{
    UnregisterClass(mWindowClass.c_str(), mInstance);
}

// run the app
int App3D::Run()
{
    if (!InitWindow())
        return 1;
    if (!InitAPI())
        return 1;
    if (!InitScene())
        return 1;
    return MsgLoop();
}

// handle input
LRESULT App3D::WndProc(HWND hWnd, UINT msg, WPARAM wParam, LPARAM lParam)
{
    switch (msg)
    {
    case WM_KEYDOWN:
        if (wParam == VK_ESCAPE)
            DestroyWindow(hWnd);
        if (wParam == VK_LEFT)
            input.left = true;
        if (wParam == VK_RIGHT) 
            input.right = true;
        if (wParam == VK_UP)
            input.up = true;
        if (wParam == VK_DOWN)
            input.down = true;
        if (wParam == 0x53)
        {
            SaveSnapshot("snapshot.bmp");
        }
        return 0;
    case WM_DESTROY:
        PostQuitMessage(0);
        return 0;
    default:
        return DefWindowProc(hWnd, msg, wParam, lParam);
    }
}

// create win32 window
bool App3D::InitWindow()
{
    WNDCLASSEX wcex;
    ZeroMemory(&wcex, sizeof(WNDCLASSEX));
    wcex.cbSize = sizeof(WNDCLASSEX);
    wcex.style = CS_HREDRAW | CS_VREDRAW;
    wcex.lpfnWndProc = MainWndProc;
    wcex.cbClsExtra = 0;
    wcex.cbWndExtra = 0;
    wcex.hInstance = mInstance;
    wcex.hIcon = LoadIcon(mInstance, MAKEINTRESOURCE(IDI_APPLICATION));
    wcex.hCursor = LoadCursor(NULL, IDC_ARROW);
    wcex.hbrBackground = (HBRUSH)(COLOR_WINDOW + 1);
    wcex.lpszMenuName = NULL;
    wcex.lpszClassName = mWindowClass.c_str();
    wcex.hIconSm = LoadIcon(mInstance, MAKEINTRESOURCE(IDI_APPLICATION));

    if (!RegisterClassEx(&wcex))
    {
        MessageBox(NULL, "Error registering class", "Error", MB_OK | MB_ICONERROR);
        return false;
    }

    RECT r = { 0, 0, mWidth, mHeight };
    AdjustWindowRect(&r, mWindowStyle, FALSE);
    unsigned int width = r.right - r.left;
    unsigned int height = r.bottom - r.top;
    unsigned int x = GetSystemMetrics(SM_CXSCREEN) / 2 - width / 2;
    unsigned int y = GetSystemMetrics(SM_CYSCREEN) / 2 - height / 2;

    mWindow = CreateWindow(
        mWindowClass.c_str(),
        mAppTitle.c_str(),
        WS_OVERLAPPEDWINDOW,
        x, y,
        width, height,
        NULL,
        NULL,
        mInstance,
        NULL
        );

    if (!mWindow)
    {
        MessageBox(NULL, "Error creating window", "Error", MB_OK | MB_ICONERROR);
        return false;
    }

    ShowWindow(mWindow, SW_SHOW);
    UpdateWindow(mWindow);

    return true;
}

// main game loop
int App3D::MsgLoop()
{
    StartTime();

    __int64 first = 0;
    __int64 second = 0;
    __int64 freq = 0;
    QueryPerformanceFrequency((LARGE_INTEGER*)&freq);

    MSG msg;
    ZeroMemory(&msg, sizeof(MSG));

    mInLoop = true;
    while (WM_QUIT != msg.message)
    {
        if (PeekMessage(&msg, NULL, NULL, NULL, PM_REMOVE))
        {
            TranslateMessage(&msg);
            DispatchMessage(&msg);
        }
        else
        {
            UpdateDeltaTime();

            QueryPerformanceCounter((LARGE_INTEGER*)&first);
            if (update)
            {
                Update();
            }
            QueryPerformanceCounter((LARGE_INTEGER*)&second);
            mUpdateTime = (float)(second - first) / (float)freq;

            QueryPerformanceCounter((LARGE_INTEGER*)&first);
            Render();
            QueryPerformanceCounter((LARGE_INTEGER*)&second);
            mRenderTime = (float)(second - first) / (float)freq;

            if (!benchmarking)
            {
                UpdateWindowTitle();
            }

            QueryPerformanceCounter((LARGE_INTEGER*)&first);
            SwapBuffer();
            QueryPerformanceCounter((LARGE_INTEGER*)&second);
            mSwapBufferTime = (float)(second - first) / (float)freq;

            UpdateTime();

            if (!Benchmark())
                DestroyWindow(mWindow);
        }
    }
    return static_cast<int>(msg.wParam);
}

void App3D::StartTime()
{
    QueryPerformanceCounter((LARGE_INTEGER*)&mPreviousCount);
    QueryPerformanceFrequency((LARGE_INTEGER*)&mFrequency);
}

void App3D::UpdateDeltaTime()
{
    QueryPerformanceCounter((LARGE_INTEGER*)&mCurrentCount);
    mDeltaTime = (float)(mCurrentCount - mPreviousCount) / (float)mFrequency;

    mElapsedTime += mDeltaTime;
    ++mDeltaFrameCount;
    if (mElapsedTime >= 1.0f)
    {
        mFPS = (float)mDeltaFrameCount;
        mElapsedTime -= 1.0f;
        mDeltaFrameCount = 0;
    }
}

void App3D::UpdateTime()
{
    mPreviousCount = mCurrentCount;
}

bool App3D::Benchmark()
{
    if (benchmarking)
    {
        if (!firstSnapshot)
        {
            CreateDirectory("Results", NULL);
            SaveSnapshot("Results/" + mBenchmarkResultName + "_start.bmp");
            firstSnapshot = true;
        }
        else
        {
            if (mFrameCount <= benchmarkFrameCount - 1)
            {
                if (mFrameTimes.size() != benchmarkFrameCount)
                    mFrameTimes.resize(benchmarkFrameCount);
                if (mUpdateTimes.size() != benchmarkFrameCount)
                    mUpdateTimes.resize(benchmarkFrameCount);
                if (mRenderTimes.size() != benchmarkFrameCount)
                    mRenderTimes.resize(benchmarkFrameCount);
                if (mSwapBufferTimes.size() != benchmarkFrameCount)
                    mSwapBufferTimes.resize(benchmarkFrameCount);

                mFrameTimes[mFrameCount] = mDeltaTime;
                mUpdateTimes[mFrameCount] = mUpdateTime;
                mRenderTimes[mFrameCount] = mRenderTime;
                mSwapBufferTimes[mFrameCount] = mSwapBufferTime;

                ++mFrameCount;

                if (mFrameCount == benchmarkFrameCount)
                {
                    std::ofstream file;
                    file.open("Results/" + mBenchmarkResultName + ".txt");
                    float deltaSum = 0;
                    float updateSum = 0;
                    float renderSum = 0;
                    float swapBufferSum = 0;
                    for (int j = 0; j < benchmarkFrameCount; ++j)
                    {
                        float delta = mFrameTimes[j];
                        float update = mUpdateTimes[j];
                        float render = mRenderTimes[j];
                        float swapBuffer = mSwapBufferTimes[j];

                        deltaSum += delta;
                        updateSum += update;
                        renderSum += render;
                        swapBufferSum += swapBuffer;

                        file << std::setfill('0') << std::setw(4) << j + 1 << " DT "
                             << std::fixed << delta << " U " << update << " R " << render
                             << " B " << swapBuffer << std::endl;
                    }
                    file << "SUM " << std::fixed << "DT " << deltaSum << " U " << updateSum
                         << " R " << renderSum << " B " << swapBufferSum << std::endl;
                    file << "FPS " << (float)benchmarkFrameCount / deltaSum << std::endl;
                    file.close();
                }
            }
            else
            {
                SaveSnapshot("Results/" + mBenchmarkResultName + "_end.bmp");
                if (saveDeviceInfo)
                {
                    SaveDeviceInfo("Results/" + mDeviceInfoFileName + ".txt");
                }
                return false;
            }
        }
    }
    return true;
}
{% endhighlight %}

      	<h3>DXApp.h</h3>
{% highlight cpp %}
#pragma once
#include "App3D.h"
#include "DXUtil.h"
#include <d3d11_2.h>
#include <d3dcompiler.h>

class DXApp : public App3D
{
public:
    DXApp(HINSTANCE hInstance);
    virtual ~DXApp();

    bool debug;

protected:
    
    ID3D11Device1*              mDevice;
    ID3D11DeviceContext1*       mDeviceContext;
    IDXGISwapChain1*            mSwapChain;
    ID3D11DepthStencilView*     mDepthStencilView;
    ID3D11RenderTargetView*     mRenderTargetView;
    D3D_DRIVER_TYPE             mDriverType;
    D3D_FEATURE_LEVEL           mFeatureLevel;
    D3D11_VIEWPORT              mViewport;

    virtual bool InitScene() = 0;
    virtual void Update() = 0;
    virtual void Render() = 0;

private:
    bool InitAPI() override;
    void UpdateWindowTitle() override;
    void SwapBuffer() override;
    void SaveSnapshot(std::string file) override;
    void SaveDeviceInfo(std::string file) override;
};


{% endhighlight %}

      	<h3>DXApp.cpp</h3>
{% highlight cpp %}
#include "DXApp.h"

DXApp::DXApp(HINSTANCE hInstance) : App3D(hInstance)
{
    mAppTitle = "DirectX App";
    mWindowClass = "DXAPPWNDCLASS";
    shaderPath = assetPath + "HLSL/";
    mDeviceInfoFileName = "dx_device_info";

    mDevice = nullptr;
    mDeviceContext = nullptr;
    mSwapChain = nullptr;
    mRenderTargetView = nullptr;

    debug = false;
}

DXApp::~DXApp()
{
    mDeviceContext->Release();
    mSwapChain->Release();
    mDevice->Release();
    mDepthStencilView->Release();
    mRenderTargetView->Release();
}

bool DXApp::InitAPI()
{
    UINT createFlags = (debug) ? D3D11_CREATE_DEVICE_DEBUG : 0;
    D3D_DRIVER_TYPE driverTypes[] =
    {
        D3D_DRIVER_TYPE_HARDWARE,
        D3D_DRIVER_TYPE_WARP,
        D3D_DRIVER_TYPE_REFERENCE,
    };
    UINT numDriverTypes = ARRAYSIZE(driverTypes);

    D3D_FEATURE_LEVEL featureLevels[] =
    {
        D3D_FEATURE_LEVEL_11_1,
        D3D_FEATURE_LEVEL_11_0,
        D3D_FEATURE_LEVEL_10_1,
        D3D_FEATURE_LEVEL_10_0,
        D3D_FEATURE_LEVEL_9_3
    };
    UINT numFeatureLevels = ARRAYSIZE(featureLevels);

    ID3D11Device* mDeviceOld = nullptr;
    ID3D11DeviceContext* mDeviceContextOld = nullptr;

    HRESULT result;
    for (unsigned int i = 0; i < numDriverTypes; ++i)
    {
        result = D3D11CreateDevice(NULL, driverTypes[i], NULL, createFlags, featureLevels, numFeatureLevels,
            D3D11_SDK_VERSION, &mDeviceOld, &mFeatureLevel, &mDeviceContextOld);

        if (result == E_INVALIDARG)
        {
            result = D3D11CreateDevice(NULL, driverTypes[i], NULL, createFlags, &featureLevels[1], numFeatureLevels - 1,
                D3D11_SDK_VERSION, &mDeviceOld, &mFeatureLevel, &mDeviceContextOld);
        }

        if (SUCCEEDED(result))
        {
            mDriverType = driverTypes[i];
            break;
        }
    }

    if (FAILED(result))
    {
        MessageBox(NULL, "Error creating D3D11 device", "Error", MB_OK | MB_ICONERROR);
        return false;
    }

    result = mDeviceOld->QueryInterface(__uuidof(ID3D11Device1), reinterpret_cast<void**>(&mDevice));
    mDeviceContextOld->QueryInterface(__uuidof(ID3D11DeviceContext1), reinterpret_cast<void**>(&mDeviceContext));

    IDXGIDevice1* dxgiDevice = nullptr;
    mDevice->QueryInterface(__uuidof(IDXGIDevice1), reinterpret_cast<void**>(&dxgiDevice));

    IDXGIAdapter* dxgiAdapter = nullptr;
    dxgiDevice->GetAdapter(&dxgiAdapter);

    IDXGIFactory2* dxgiFactory = nullptr;
    dxgiAdapter->GetParent(__uuidof(IDXGIFactory2), reinterpret_cast<void**>(&dxgiFactory));

    DXGI_SWAP_CHAIN_DESC1 sd;
    ZeroMemory(&sd, sizeof(sd));
    sd.Width = mWidth;
    sd.Height = mHeight;
    sd.Format = DXGI_FORMAT_R8G8B8A8_UNORM;
    sd.SampleDesc.Count = 1;
    sd.SampleDesc.Quality = 0;
    sd.BufferUsage = DXGI_USAGE_RENDER_TARGET_OUTPUT;
    sd.BufferCount = 1;

    dxgiFactory->CreateSwapChainForHwnd(mDevice, mWindow, &sd, NULL, NULL, &mSwapChain);

    ID3D11Texture2D* backBuffer;
    mSwapChain->GetBuffer(0, __uuidof(ID3D11Texture2D), reinterpret_cast<void**>(&backBuffer));

    mDevice->CreateRenderTargetView(backBuffer, NULL, &mRenderTargetView);

    D3D11_TEXTURE2D_DESC depthStencilDesc;
    ZeroMemory(&depthStencilDesc, sizeof(depthStencilDesc));
    depthStencilDesc.Width = mWidth;
    depthStencilDesc.Height = mHeight;
    depthStencilDesc.MipLevels = 1;
    depthStencilDesc.ArraySize = 1;
    depthStencilDesc.Format = DXGI_FORMAT_D24_UNORM_S8_UINT;
    depthStencilDesc.SampleDesc.Count = 1;
    depthStencilDesc.SampleDesc.Quality = 0;
    depthStencilDesc.Usage = D3D11_USAGE_DEFAULT;
    depthStencilDesc.BindFlags = D3D11_BIND_DEPTH_STENCIL;
    depthStencilDesc.CPUAccessFlags = 0;
    depthStencilDesc.MiscFlags = 0;

    ID3D11Texture2D* depthStencilBuffer;
    mDevice->CreateTexture2D(&depthStencilDesc, NULL, &depthStencilBuffer);
    mDevice->CreateDepthStencilView(depthStencilBuffer, NULL, &mDepthStencilView);

    mDeviceContext->OMSetRenderTargets(1, &mRenderTargetView, mDepthStencilView);

    mViewport.Width = (FLOAT)mWidth;
    mViewport.Height = (FLOAT)mHeight;
    mViewport.MinDepth = 0.0f;
    mViewport.MaxDepth = 1.0f;
    mViewport.TopLeftX = 0;
    mViewport.TopLeftY = 0;
    mDeviceContext->RSSetViewports(1, &mViewport);

    mDeviceOld->Release();
    mDeviceContextOld->Release();
    dxgiDevice->Release();
    dxgiAdapter->Release();
    dxgiFactory->Release();
    backBuffer->Release();
    depthStencilBuffer->Release();

    return true;
}

void DXApp::UpdateWindowTitle()
{
    std::stringstream ss;
    ss << mAppTitle << " | FPS " << mFPS;
    SetWindowText(mWindow, ss.str().c_str());
}

void DXApp::SwapBuffer()
{
    mSwapChain->Present(0, 0);
}

void DXApp::SaveSnapshot(std::string filePath)
{
    ID3D11Texture2D *backBuffer;
    mSwapChain->GetBuffer(0, __uuidof(ID3D11Texture2D), reinterpret_cast<void **>(&backBuffer));

    D3D11_TEXTURE2D_DESC desc;
    ZeroMemory(&desc, sizeof(desc));
    backBuffer->GetDesc(&desc);
    desc.Format = DXGI_FORMAT_R8G8B8A8_UNORM;
    desc.Usage = D3D11_USAGE_STAGING;
    desc.BindFlags = 0;
    desc.CPUAccessFlags = D3D11_CPU_ACCESS_READ | D3D11_CPU_ACCESS_WRITE;

    ID3D11Texture2D *newTex;
    mDevice->CreateTexture2D(&desc, NULL, &newTex);
    mDeviceContext->CopyResource(newTex, backBuffer);

    D3D11_MAPPED_SUBRESOURCE res;
    ZeroMemory(&res, sizeof(res));
    mDeviceContext->Map(newTex, 0, D3D11_MAP_READ, 0, &res);

    byte *bmpTempBuffer = reinterpret_cast<byte *>(res.pData);
    if (!bmpTempBuffer) return;

    byte *bmpBuffer = new byte[mWidth * mHeight * 3];

    for (int y = 0; y < mHeight; ++y)
    {
        for (int x = 0; x < mWidth; ++x)
        {
            int i = (x + (y * mWidth)) * 3;
            int iInv = (x + ((mWidth - y - 1) * mWidth)) * 3;
            int c = iInv / 3;
            bmpBuffer[i + 0] = bmpTempBuffer[iInv + 2 + c];
            bmpBuffer[i + 1] = bmpTempBuffer[iInv + 1 + c];
            bmpBuffer[i + 2] = bmpTempBuffer[iInv + 0 + c];
        }
    }

    mDeviceContext->Unmap(newTex, 0);
    bmpTempBuffer = nullptr;
    backBuffer->Release();
    newTex->Release();

    std::ofstream file(filePath, std::ios::out | std::ios::binary);
    if (!file.is_open()) return;

    BITMAPFILEHEADER bitmapFileHeader;
    bitmapFileHeader.bfType = 0x4D42; //"BM"
    bitmapFileHeader.bfSize = mWidth * mHeight * 3;
    bitmapFileHeader.bfReserved1 = 0;
    bitmapFileHeader.bfReserved2 = 0;
    bitmapFileHeader.bfOffBits = sizeof(BITMAPFILEHEADER) + sizeof(BITMAPINFOHEADER);

    BITMAPINFOHEADER bitmapInfoHeader;
    bitmapInfoHeader.biSize = sizeof(BITMAPINFOHEADER);
    bitmapInfoHeader.biWidth = mWidth;
    bitmapInfoHeader.biHeight = mHeight;
    bitmapInfoHeader.biPlanes = 1;
    bitmapInfoHeader.biBitCount = 24;
    bitmapInfoHeader.biCompression = BI_RGB;
    bitmapInfoHeader.biSizeImage = 0;
    bitmapInfoHeader.biXPelsPerMeter = 0; // ?
    bitmapInfoHeader.biYPelsPerMeter = 0; // ?
    bitmapInfoHeader.biClrUsed = 0;
    bitmapInfoHeader.biClrImportant = 0;

    file.write(reinterpret_cast<char *>(&bitmapFileHeader), sizeof(bitmapFileHeader));
    file.write(reinterpret_cast<char *>(&bitmapInfoHeader), sizeof(bitmapInfoHeader));
    file.write(reinterpret_cast<char *>(bmpBuffer), mWidth * mHeight * 3);
    file.close();

    delete[] bmpBuffer;
}

void DXApp::SaveDeviceInfo(std::string filePath)
{
    D3D_FEATURE_LEVEL featureLevel = mDevice->GetFeatureLevel();
    std::string featureLevelString = "Direct3D ";

    switch (featureLevel)
    {
    case D3D_FEATURE_LEVEL_9_1:
        featureLevelString += "9.1";
        break;
    case D3D_FEATURE_LEVEL_9_2:
        featureLevelString += "9.2";
        break;
    case D3D_FEATURE_LEVEL_9_3:
        featureLevelString += "9.3";
        break;
    case D3D_FEATURE_LEVEL_10_0:
        featureLevelString += "10.";
        break;
    case D3D_FEATURE_LEVEL_10_1:
        featureLevelString += "10.1";
        break;
    case D3D_FEATURE_LEVEL_11_0:
        featureLevelString += "11.0";
        break;
    case D3D_FEATURE_LEVEL_11_1:
        featureLevelString += "11.1";
        break;
    }

    std::ofstream file(filePath);
    file << featureLevelString << std::endl;
    file.close();
}
{% endhighlight %}

      	<h3>GLApp.h</h3>
{% highlight cpp %}
#pragma once
#include "App3D.h"
#include <glew.h>
#include <wglew.h>
#include <gl\GL.h>
#include <gl\GLU.h>
#include <glext.h>
#include <wglext.h>
#include "GLUtil.h"

class GLApp : public App3D
{
public:
    GLApp(HINSTANCE hInstance);
    virtual ~GLApp();

    bool vSync;

protected:
    HDC     mDeviceContext;
    HGLRC   mGLRenderContext;

    virtual bool InitScene() = 0;
    virtual void Update() = 0;
    virtual void Render() = 0;

private:
    PFNWGLSWAPINTERVALEXTPROC           wglSwapIntervalEXT = nullptr;
    PFNWGLGETSWAPINTERVALEXTPROC        wglGetSwapIntervalEXT = nullptr;
    PFNWGLCREATECONTEXTATTRIBSARBPROC   wglCreateContextAttribsARB = nullptr;

    bool InitAPI() override;
    void UpdateWindowTitle() override;
    void SwapBuffer() override;
    void SaveSnapshot(std::string file) override;
    void SaveDeviceInfo(std::string file) override;
    
    bool WGLExtSupported(std::string extName);
    void SetVsync();
    void PrintVersion();
};


{% endhighlight %}

      	<h3>GLApp.cpp</h3>
{% highlight cpp %}
#include "GLApp.h"

GLApp::GLApp(HINSTANCE hInstance) : App3D(hInstance)
{
    mAppTitle = "OpenGL App";
    mWindowClass = "GLAPPWNDCLASS";
    shaderPath = assetPath + "GLSL/";
    mDeviceInfoFileName = "gl_device_info";

    mDeviceContext = nullptr;
    mGLRenderContext = nullptr;

    vSync = false;
}

GLApp::~GLApp()
{
    wglMakeCurrent(NULL, NULL);
    wglDeleteContext(mGLRenderContext);
    ReleaseDC(mWindow, mDeviceContext);
}

bool GLApp::InitAPI()
{
    mDeviceContext = GetDC(mWindow);

    PIXELFORMATDESCRIPTOR pfd;
    ZeroMemory(&pfd, sizeof(PIXELFORMATDESCRIPTOR));
    pfd.nSize = sizeof(PIXELFORMATDESCRIPTOR);
    pfd.nVersion = 1;
    pfd.dwFlags = PFD_DRAW_TO_WINDOW | PFD_SUPPORT_OPENGL | PFD_DOUBLEBUFFER;
    pfd.iPixelType = PFD_TYPE_RGBA;
    pfd.cColorBits = 32;
    pfd.cDepthBits = 24;
    pfd.cStencilBits = 8;

    int format = ChoosePixelFormat(mDeviceContext, &pfd);
    if (!SetPixelFormat(mDeviceContext, format, &pfd))
    {
        MessageBox(NULL, "Error setting pixel format", "Error", MB_OK | MB_ICONERROR);
        return false;
    }

    mGLRenderContext = wglCreateContext(mDeviceContext);
    if (!wglMakeCurrent(mDeviceContext, mGLRenderContext))
    {
        MessageBox(NULL, "Error creating and activatign render context", "Error", MB_OK | MB_ICONERROR);
        return false;
    }

    if (glewInit())
    {
        MessageBox(NULL, "Error initializing GLEW", "Error", MB_OK | MB_ICONERROR);
        return false;
    }

    glViewport(0, 0, (GLsizei)mWidth, (GLsizei)mHeight);

    SetVsync();

    return true;
}

void GLApp::UpdateWindowTitle()
{
    std::stringstream ss;
    ss << mAppTitle << " | FPS " << mFPS;
    SetWindowText(mWindow, ss.str().c_str());
}

void GLApp::SwapBuffer()
{
    SwapBuffers(mDeviceContext);
}

void GLApp::SaveSnapshot(std::string filePath)
{
    byte *bmpBuffer = new byte[mWidth * mHeight * 3];
    if (!bmpBuffer) return;

    glReadBuffer(GL_FRONT);
    glReadPixels(0, 0, mWidth, mHeight, GL_BGR, GL_UNSIGNED_BYTE, bmpBuffer);

    std::ofstream file(filePath, std::ios::out | std::ios::binary);
    if (!file.is_open()) return;

    BITMAPFILEHEADER bitmapFileHeader;
    bitmapFileHeader.bfType = 0x4D42; //"BM"
    bitmapFileHeader.bfSize = mWidth * mHeight * 3;
    bitmapFileHeader.bfReserved1 = 0;
    bitmapFileHeader.bfReserved2 = 0;
    bitmapFileHeader.bfOffBits = sizeof(BITMAPFILEHEADER) + sizeof(BITMAPINFOHEADER);

    BITMAPINFOHEADER bitmapInfoHeader;
    bitmapInfoHeader.biSize = sizeof(BITMAPINFOHEADER);
    bitmapInfoHeader.biWidth = mWidth;
    bitmapInfoHeader.biHeight = mHeight;
    bitmapInfoHeader.biPlanes = 1;
    bitmapInfoHeader.biBitCount = 24;
    bitmapInfoHeader.biCompression = BI_RGB;
    bitmapInfoHeader.biSizeImage = 0;
    bitmapInfoHeader.biXPelsPerMeter = 0; // ?
    bitmapInfoHeader.biYPelsPerMeter = 0; // ?
    bitmapInfoHeader.biClrUsed = 0;
    bitmapInfoHeader.biClrImportant = 0;

    file.write(reinterpret_cast<char *>(&bitmapFileHeader), sizeof(bitmapFileHeader));
    file.write(reinterpret_cast<char *>(&bitmapInfoHeader), sizeof(bitmapInfoHeader));
    file.write(reinterpret_cast<char *>(bmpBuffer), mWidth * mHeight * 3);
    file.close();

    delete[] bmpBuffer;
}

void GLApp::SaveDeviceInfo(std::string filePath)
{
    GLint majorVersion = 0;
    GLint minorVersion = 0;
    glGetIntegerv(GL_MAJOR_VERSION, &majorVersion);
    glGetIntegerv(GL_MINOR_VERSION, &minorVersion);
    std::string glslVersion = reinterpret_cast<const char*>(glGetString(GL_SHADING_LANGUAGE_VERSION));
    std::string vendor = reinterpret_cast<const char*>(glGetString(GL_VENDOR));
    std::string renderer = reinterpret_cast<const char*>(glGetString(GL_RENDERER));

    std::ofstream file(filePath);
    file << "OpenGL " << std::to_string(majorVersion) << "." << std::to_string(minorVersion) << std::endl;
    file << "GLSL " << glslVersion << std::endl;
    file << "Vendor: " << vendor << std::endl;
    file << "Renderer: " << renderer << std::endl;
    file.close();
}

// check if extension is supported
bool GLApp::WGLExtSupported(std::string extName)
{
    PFNWGLGETEXTENSIONSSTRINGEXTPROC _wglGetExtensionsStringEXT = NULL;
    _wglGetExtensionsStringEXT = (PFNWGLGETEXTENSIONSSTRINGEXTPROC)wglGetProcAddress("wglGetExtensionsStringEXT");
    return !(strstr(_wglGetExtensionsStringEXT(), extName.c_str()) == NULL);
}

void GLApp::SetVsync()
{
    if (WGLExtSupported("WGL_EXT_swap_control"))
    {
        wglSwapIntervalEXT = (PFNWGLSWAPINTERVALEXTPROC)wglGetProcAddress("wglSwapIntervalEXT");
        wglGetSwapIntervalEXT = (PFNWGLGETSWAPINTERVALEXTPROC)wglGetProcAddress("wglGetSwapIntervalEXT");
        wglSwapIntervalEXT(vSync);
    }
}
{% endhighlight %}

    </div>
</div>