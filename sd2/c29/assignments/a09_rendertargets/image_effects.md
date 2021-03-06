Effect Framework
======

## End Goal
```cpp
void Game::Render()
{
   // I want to render to the backbuffer (the image that
   // will present to the user), but can't
   // use that for image effects...
   // ...so get a texture I can use that matches it.
   Texture* backbuffer = m_renderer->GetFrameBackbuffer(); 
   Texture* output = m_renderer->AcquireRenderTargetMatching( backbuffer ); 

   // Set that as the output - and draw
   m_camera->SetColorTarget( output ); 
   m_renderer->BeginCamera( m_camera ); 
   // render world...
   m_renderer->EndCamera(); 

   // Now, we want to apply an effect to the game given a material
   Texture* temp = m_renderer->AcquireRenderTargetMatching( output ); 
   m_renderer->ApplyEffect( temp, output, m_tonemap_material ); 

   // The final output is now in temp, so copy it back
   // to the backbuffer so we can present
   m_renderer->CopyTexture( backbuffer, temp );

   // Release the two temporary textures back to the pool
   m_renderer->ReleaseRenderTarget( temp ); 
   m_renderer->ReleaseRenderTarget( output ); 

   // Print out how many are alive as a debug step
   // to make sure we're not leaking (for this 
   // demo, we'll have at most 2)
   DebugRenderScreenText( "Pool Size: %u free of %u total", 
      m_renderer->GetFreeRenderTargetCount(), 
      m_renderer->GetTotalRenderTargetCount() ); 

   // Present backbuffer to the user...
   // ...this is happening in `RenderContext::EndFrame()
   // m_renderer->Present(); 
}
```

## Render Context Methods
```cpp
class RenderContext
{
   public:
      // old things we should already have
      Texture* GetFrameBackbuffer(); 
      void Present();

      // NEW things
      // TexturePool Things
      Texture* AcquireRenderTargetMatching( Texture* tex ); // user should release with call below
      void ReleaseRenderTarget( Texture* tex );             // recycles texture to be used again
      Texture* CreateRenderTarget( int width, int height ); // returned object should be deleted

      // Debug Things
      int GetTotalRenderTargetCount(); // how many temporary textures have we made total
      int GetFreeRenderTargetCount(); // how many are currently in the pool

      // Effect Framework
      void ApplyEffect( Texture* dst, Texture* src, Material* mat ); 
      void CopyTexture( Texture* dst, Texture* src ); 

};
```

### Steps
- [ ] Be able to render to a custom render target
    - [ ] Be able to create a render target
    - [ ] Do the render target pool implementation
    - [ ] Verify using `RenderDoc` or equivalent
- [ ] Be able to copy textures -> `CopyTexture`
- [ ] Verify you can now see things again
- [ ] Implement `ApplyEffect`
    - [ ] Some variants if you don't have materials yet

### Creating the Render Target
Same as before - but notice I make the texture as
usable as both a RenderTarget and a Shader Resource. 

```cpp
Texture* RenderContext::CreateRenderTarget( IntVec2 texelSize )
{
   D3D11_TEXTURE2D_DESC desc;
   desc.Width = texelSize.x;
   desc.Height = texelSize.y;
   desc.MipLevels = 1;
   desc.ArraySize = 1;
   desc.Format = DXGI_FORMAT_R8G8B8A8_UNORM;
   desc.SampleDesc.Count = 1; // MSAA
   desc.SampleDesc.Quality = 0;
   desc.Usage = D3D11_USAGE_DEFAULT; // if you do mip-chians, we need this to be GPU/DEFAULT
   desc.BindFlags = D3D11_BIND_RENDER_TARGET | D3D11_BIND_SHADER_RESOURCE;
   desc.CPUAccessFlags = 0;
   desc.MiscFlags = 0;

   //DirectX Creation
   ID3D11Texture2D* texHandle = nullptr;
   m_device->CreateTexture2D( &desc, nullptr, &texHandle );

   Texture* newTexture = new Texture( this, texHandle, nullptr );
   return newTexture;
}
```

### Copying a Texture
```cpp
void RenderContext::CopyTexture( Texture* dst, Texture* src )
{
   m_context->CopyResource( dst->GetHandle(), src->GetHandle() );  
}
```

### `Texture` Pooling
```cpp
Texture* RenderContext::AcquireRenderTargetMatching( Texture* texture )
{
   IntVec2 size = texture->GetTexelSize(); 

   for (int i = 0; i < m_renderTargetPool.size(); ++i) 
   {
      Texture* rt = m_renderTargetPool[i]; 
      if (rt->GetTexelSize() == size)
      {
         // fast remove at index
         m_renderTargetPool[i] = m_renderTargetPool[ m_renderTargetPool.size() - 1 ]; 
         m_renderTargetPool.pop_back(); 

         // return the object from the pool
         return rt; 
      }
   }

   // nothing in the pool that matches - make a new one
   ++m_totalRenderTargetMade;
   Texture* newRenderTarget = CreateRenderTarget( size ); 
   return newRenderTarget; 
}

void RenderContext::ReleaseRenderTarget( Texture* tex )
{
   m_renderTargetPool.push_back( tex ); 
}
```

And don't forget to free these up when the system shuts down...


### `RenderContext::ApplyEffect`
Effects happen OUTSIDE of `BeingCamera` and `EndCamera`, so you may want to confirm you do
not have an active camera.

What we want to finally get to is a...

```cpp
void RenderContext::ApplyEffect( Texture* dst, Texture* src, Material* mat ); 
```

This will be implemented using the following two methods if a material class is not available (or if you just don't want to use it...)

```cpp
void RenderContext::BeginEffect( Texture* dst, Texture* src, Shader* shader ); 
void RenderContext::EndEffect(); 
```


#### Begin Efffect
An effect at the most basic level is just rendering to a texture "something" (the shader), usually
using another `src` texture, plus other data, as the input.

So this would need to...
- [ ] Set a render target to be the `dst`
- [ ] Setup to render to it (we will use a camera here to handle most of the details, but you 
      could also do this manually). 
- [ ] Bind our shader.
- [ ] Bind the input data (`src`) texture in this case... though this is mostly for 
      convenience as the user could have bound this themselves before `EndEffect`

```cpp
void RenderContext::BeginEffect( Texture* dst, Texture* src, Shader* shader ) 
{
   // this should have identiy project, view, and model at this point
   // Could just do this with "BindColorTarget" if available
   m_effectCamera->SetColorTarget( dst ); 

   BeginCamera( *m_effectCamera ); 
   BindShader( shader ); 
   BindTexture( src ); 
}
```

#### End Effect

Ending an effect is going to draw the quad (in this case, a "full screen triangle").

Before `EndEffect` is called, it is up to the user to set any other data this effect may need, such
as other uniform values or supplementary textures. 

```cpp
void RenderContext::EndEffect()
{
   m_context->IASetPrimitiveTopology( D3D11_PRIMITIVE_TOPOLOGY_TRIANGLELIST ); 
   m_context->Draw( 3, 0 );
   EndCamera( *m_effectCamera ); 
}
```

### Apply Effect
Now `ApplyEffect` is just using `Begin` and `End`, since the material can
store all the suppementary effect data needed.  So once you have materials
you can move into a slightly simpler syntax, but still support the non-material version!

```cpp
void RenderContext::ApplyEffect( Texture* dst, Texture* src, Material* mat )
{
   BeginEffect(...);
   // bind material data
   EndEffect(); 
}
```

### The Full Screen Triangle

When rendering, we'll use baked in vertex data (hence the `Draw(3,0)` with no vertex data bound), 
similar to the very first shader we did in class.

See [./ImageEffect.hlsl](./ImageEffect.hlsl) for an example.  


------
## Camera Effect Framework
This is completely optional - and just brings this more in line with
 `Unity` or `Unreal`'s' interface, where you attach effects to a camera
 and it automatically applies

```cpp
void Camera
{
   public:
      void AddEffect( Material* mat, int sortOrder = 0 ); 
      void RemoveEffect( Material* mat ); 
}
```

Idea is during `RenderContext::EndCamera`, we will automatically
apply all effects on the camera by creating two temporary textures
matching the camera's output, and then just walks the `effect stack` 
applying effects, finally copying the final result back to the original
output.  

It is a way to automate the steps demonstrated in `Game::Render` above, but
would suggest only doing so once you have the proper infastructure in place 
instead of only letting it exist on the camera.

