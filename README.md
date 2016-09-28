GLSurfaceView 结合 OpenGLES使用步骤(个人总结，如果觉得不对，欢迎指正)：
1.找到view,初始化
    mEffectView = (GLSurfaceView) view.findViewById(R.id.effectsview);
    mEffectView.setEGLContextClientVersion(2);// 用OpenGLES 2.0
    mEffectView.setRenderer(this);// Renderer
    mEffectView.setRenderMode(GLSurfaceView.RENDERMODE_WHEN_DIRTY);// 只在绘制数据发生改变时才绘制view
2. 实现 Renderer 的三个函数
      @Override
      public void onSurfaceCreated(GL10 gl, EGLConfig eglConfig) {
          // 仅调用一次，用于设置view的OpenGLES环境
      }
      // 如果view的几何形状发生变化了就调用，例如当竖屏变为横屏时
      @Override
      public void onSurfaceChanged(GL10 gl, int width, int height) {

      }
      // 每次View被重绘时被调用,重点
      @Override
      public void onDrawFrame(GL10 gl) {

      }
3.  用来显示图片的是 TextureRenderer   onDrawFrame 里 初始化，只一次
     这里 create a vertex shader   create a fragment shader
     int shader = GLES20.glCreateShader(shaderType);// create a shader object and return a reference
     GLES20.glShaderSource(shader, source);// associate the shader code (source) with the shader.
     GLES20.glCompileShader(shader);// compile the shader code
     GLES20.glGetShaderiv(shader, GLES20.GL_COMPILE_STATUS, compiled, 0);

创建好后
     GLES20.glAttachShader(program, vertexShader);// attach the shaders to the program.
     GLES20.glAttachShader(program, pixelShader);
     GLES20.glLinkProgram(program); // link the program
     GLES20.glGetProgramiv(program, GLES20.GL_LINK_STATUS, linkStatus, 0);

     // Bind attributes and uniforms
     // get a handle to the constant uTexture mentioned in the fragment shader code.
     mTexSamplerHandle = GLES20.glGetUniformLocation(mProgram,"tex_sampler");
     // get a handle to the variables a_position and a_texcoord mentioned in the vertex shader code.
     mTexCoordHandle = GLES20.glGetAttribLocation(mProgram, "a_texcoord");
     mPosCoordHandle = GLES20.glGetAttribLocation(mProgram, "a_position");

       /* ByteBuffer.allocateDirect:   create buffer
        * ByteBuffer.nativeOrder:       determine the byte order
        * asFloatBuffer:                convert the ByteBuffer instance into a FloatBuffer
        * put:                          load the array into the buffer
        * position:                     make sure that the buffer is read from the beginning
        * */
        mTexVertices = ByteBuffer.allocateDirect(
                TEX_VERTICES.length * FLOAT_SIZE_BYTES)
                .order(ByteOrder.nativeOrder()).asFloatBuffer();
        mTexVertices.put(TEX_VERTICES).position(0);
        mPosVertices = ByteBuffer.allocateDirect(
                POS_VERTICES.length * FLOAT_SIZE_BYTES)
                .order(ByteOrder.nativeOrder()).asFloatBuffer();
        mPosVertices.put(POS_VERTICES).position(0);
到这里，GLES20与 TextureRenderer 就绑定好了

4.onDrawFrame 里 GLES20 初始化，只一次
        // Generate textures
        GLES20.glGenTextures(2, mTextures, 0);//  initialize the array

        // Upload bitmap to texture
        GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, mTextures[0]); // activate the texture at index 0
        GLUtils.texImage2D(GLES20.GL_TEXTURE_2D, 0, bitmap, 0);

        // set various properties that decide how the texture is rendered
        GLES20.glTexParameteri(GLES20.GL_TEXTURE_2D,
                GLES20.GL_TEXTURE_MAG_FILTER, GLES20.GL_LINEAR);
        GLES20.glTexParameteri(GLES20.GL_TEXTURE_2D,
                GLES20.GL_TEXTURE_MIN_FILTER, GLES20.GL_LINEAR);
        GLES20.glTexParameteri(GLES20.GL_TEXTURE_2D, GLES20.GL_TEXTURE_WRAP_S,
                GLES20.GL_CLAMP_TO_EDGE);
        GLES20.glTexParameteri(GLES20.GL_TEXTURE_2D, GLES20.GL_TEXTURE_WRAP_T,
                GLES20.GL_CLAMP_TO_EDGE);
至此，GLES20初始化完成，图片也设置进去了

5.需要显示出来
        // Bind default FBO ,create a named frame buffer object
        GLES20.glBindFramebuffer(GLES20.GL_FRAMEBUFFER, 0);

        // Use our shader program ,using the program we just linked.
        GLES20.glUseProgram(mProgram);
        // Set viewport
        GLES20.glViewport(0, 0, mViewWidth, mViewHeight);
        // Disable blending
        GLES20.glDisable(GLES20.GL_BLEND);// to disable the blending of colors while rendering

        // Set the vertex attributes
        /*
        * glVertexAttribPointer  associate the aPosition  handles with the textureBuffer
        * */
        GLES20.glVertexAttribPointer(mTexCoordHandle, 2, GLES20.GL_FLOAT, false,
                0, mTexVertices);
        GLES20.glEnableVertexAttribArray(mTexCoordHandle);

        GLES20.glVertexAttribPointer(mPosCoordHandle, 2, GLES20.GL_FLOAT, false,
                0, mPosVertices);
        GLES20.glEnableVertexAttribArray(mPosCoordHandle);

        // Set the input texture
        GLES20.glActiveTexture(GLES20.GL_TEXTURE0);
        // glBindTexture  bind the texture (passed as an argument to the draw method) to the fragment shader.
        GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, texId);
        GLES20.glUniform1i(mTexSamplerHandle, 0);

        // Draw
        GLES20.glClearColor(0.0f, 0.0f, 0.0f, 1.0f);
        // glClear :  Clear the contents of the GLSurfaceView
        GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT);
        // glDrawArrays : draw the two triangles (and thus the square).
        GLES20.glDrawArrays(GLES20.GL_TRIANGLE_STRIP, 0, 4);
done.
6.当滤镜改变时，手动通知GLSurfaceView  mEffectView.requestRender();// 手动调用 onDrawFrame
    // apply(int inputTexId, int width, int height, int outputTexId)
    mEffect.apply(mTextures[0], mImageWidth, mImageHeight, mTextures[1]);