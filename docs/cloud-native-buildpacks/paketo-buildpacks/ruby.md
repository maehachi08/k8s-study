# Paketo BuildpacksでRuby on RailsアプリケーションをOCI imageにbuildする

## 参考

- https://paketo.io/docs/reference/ruby-reference/
- https://paketo.io/docs/howto/ruby/
- https://github.com/paketo-buildpacks/ruby
- https://github.com/paketo-buildpacks/full-builder

!!! warning
    `mysql2` library のようにnative extentionを一緒にインストールする必要のあるgem libraryをインストールするためには full builder image(`paketobuildpacks/builder:full`) を指定する必要があります。
    - https://github.com/paketo-buildpacks/ruby/issues/862

    ```
    $ pack build puma-sample --buildpack paketo-buildpacks/ruby --builder paketobuildpacks/builder:full
    ```

    <details><summary>`paketobuildpacks/builder:base` では以下エラーとなりました</summary>
    ```
          Gem::Ext::BuildError: ERROR: Failed to build gem native extension.
    
          current directory:
          /layers/paketo-buildpacks_bundle-install/launch-gems/ruby/3.2.0/gems/mysql2-0.5.5/ext/mysql2
          /layers/paketo-buildpacks_mri/mri/bin/ruby -I
          /layers/paketo-buildpacks_mri/mri/lib/ruby/3.2.0 extconf.rb
          checking for rb_absint_size()... yes
          checking for rb_absint_singlebit_p()... yes
          checking for rb_gc_mark_movable()... yes
          checking for rb_wait_for_single_fd()... yes
          checking for rb_enc_interned_str() in ruby.h... yes
          *** extconf.rb failed ***
          Could not create Makefile due to some reason, probably lack of necessary
          libraries and/or headers.  Check the mkmf.log file for more details.  You may
          need configuration options.
    
          Provided configuration options:
            --with-opt-dir
            --without-opt-dir
            --with-opt-include
            --without-opt-include=${opt-dir}/include
            --with-opt-lib
            --without-opt-lib=${opt-dir}/lib
            --with-make-prog
            --without-make-prog
            --srcdir=.
            --curdir
            --ruby=/layers/paketo-buildpacks_mri/mri/bin/$(RUBY_BASE_NAME)
            --with-openssl-dir
            --without-openssl-dir
            --with-mysql-dir
            --without-mysql-dir
            --with-mysql-include
            --without-mysql-include=${mysql-dir}/include
            --with-mysql-lib
            --without-mysql-lib=${mysql-dir}/lib
            --with-mysql-config
            --without-mysql-config
            --with-mysqlclient-dir
            --without-mysqlclient-dir
            --with-mysqlclient-include
            --without-mysqlclient-include=${mysqlclient-dir}/include
            --with-mysqlclient-lib
            --without-mysqlclient-lib=${mysqlclient-dir}/lib
            --with-mysqlclientlib
            --without-mysqlclientlib
          /layers/paketo-buildpacks_mri/mri/lib/ruby/3.2.0/mkmf.rb:1083:in `block in
          find_library': undefined method `split' for nil:NilClass (NoMethodError)
    
              paths = paths.flat_map {|path| path.split(File::PATH_SEPARATOR)}
                                                 ^^^^^^
            from /layers/paketo-buildpacks_mri/mri/lib/ruby/3.2.0/mkmf.rb:1083:in `each'
          from /layers/paketo-buildpacks_mri/mri/lib/ruby/3.2.0/mkmf.rb:1083:in
          `flat_map'
          from /layers/paketo-buildpacks_mri/mri/lib/ruby/3.2.0/mkmf.rb:1083:in
          `find_library'
            from extconf.rb:131:in `<main>'
    
          To see why this extension failed to compile, please check the mkmf.log which can
          be found here:
    
          /layers/paketo-buildpacks_bundle-install/launch-gems/ruby/3.2.0/extensions/x86_64-linux/3.2.0-static/mysql2-0.5.5/mkmf.log
    
          extconf failed, exit code 1
    
          Gem files will remain installed in
          /layers/paketo-buildpacks_bundle-install/launch-gems/ruby/3.2.0/gems/mysql2-0.5.5
          for inspection.
          Results logged to
          /layers/paketo-buildpacks_bundle-install/launch-gems/ruby/3.2.0/extensions/x86_64-linux/3.2.0-static/mysql2-0.5.5/gem_make.out
    
          /layers/paketo-buildpacks_mri/mri/lib/ruby/3.2.0/rubygems/ext/builder.rb:102:in
          `run'
          /layers/paketo-buildpacks_mri/mri/lib/ruby/3.2.0/rubygems/ext/ext_conf_builder.rb:28:in
          `build'
          /layers/paketo-buildpacks_mri/mri/lib/ruby/3.2.0/rubygems/ext/builder.rb:170:in
          `build_extension'
          /layers/paketo-buildpacks_mri/mri/lib/ruby/3.2.0/rubygems/ext/builder.rb:204:in
          `block in build_extensions'
          /layers/paketo-buildpacks_mri/mri/lib/ruby/3.2.0/rubygems/ext/builder.rb:201:in
          `each'
          /layers/paketo-buildpacks_mri/mri/lib/ruby/3.2.0/rubygems/ext/builder.rb:201:in
          `build_extensions'
          /layers/paketo-buildpacks_mri/mri/lib/ruby/3.2.0/rubygems/installer.rb:843:in
          `build_extensions'
          /layers/paketo-buildpacks_bundler/bundler/gems/bundler-2.4.6/lib/bundler/rubygems_gem_installer.rb:72:in
          `build_extensions'
          /layers/paketo-buildpacks_bundler/bundler/gems/bundler-2.4.6/lib/bundler/rubygems_gem_installer.rb:28:in
          `install'
          /layers/paketo-buildpacks_bundler/bundler/gems/bundler-2.4.6/lib/bundler/source/rubygems.rb:200:in
          `install'
          /layers/paketo-buildpacks_bundler/bundler/gems/bundler-2.4.6/lib/bundler/installer/gem_installer.rb:54:in
          `install'
          /layers/paketo-buildpacks_bundler/bundler/gems/bundler-2.4.6/lib/bundler/installer/gem_installer.rb:16:in
          `install_from_spec'
          /layers/paketo-buildpacks_bundler/bundler/gems/bundler-2.4.6/lib/bundler/installer/parallel_installer.rb:155:in
          `do_install'
          /layers/paketo-buildpacks_bundler/bundler/gems/bundler-2.4.6/lib/bundler/installer/parallel_installer.rb:146:in
          `block in worker_pool'
          /layers/paketo-buildpacks_bundler/bundler/gems/bundler-2.4.6/lib/bundler/worker.rb:62:in
          `apply_func'
          /layers/paketo-buildpacks_bundler/bundler/gems/bundler-2.4.6/lib/bundler/worker.rb:57:in
          `block in process_queue'
          /layers/paketo-buildpacks_bundler/bundler/gems/bundler-2.4.6/lib/bundler/worker.rb:54:in
          `loop'
          /layers/paketo-buildpacks_bundler/bundler/gems/bundler-2.4.6/lib/bundler/worker.rb:54:in
          `process_queue'
          /layers/paketo-buildpacks_bundler/bundler/gems/bundler-2.4.6/lib/bundler/worker.rb:90:in
          `block (2 levels) in create_threads'
    
          An error occurred while installing mysql2 (0.5.5), and Bundler cannot continue.
    
          In Gemfile:
            mysql2
    failed to execute bundle install output:
    
    error: exit status 5
    ERROR: failed to build: exit status 1
    ERROR: failed to build: executing lifecycle: failed with status code: 51
    ```
    </details>

## image build

1. build
    <details><summary>`paketobuildpacks/builder:full` builderでrails appのimage作成</summary>
    ```
    $ pack build puma-sample --buildpack paketo-buildpacks/ruby --builder paketobuildpacks/builder:full
    full: Pulling from paketobuildpacks/builder
    e58c18cab017: Already exists
    
    041be6a6e80e: Pull complete
    4612107b91b7: Pull complete
    273978fd99f5: Pull complete
    4f4fb700ef54: Pull complete
    Digest: sha256:6deb1900981341c9e0dd66537b20cb64fdb7f26111a46ceb3681ba6fdf9dfea0
    Status: Downloaded newer image for paketobuildpacks/builder:full
    full-cnb: Pulling from paketobuildpacks/run
    e58c18cab017: Already exists
    e5e9174b359f: Already exists
    5e91d2c23726: Pull complete
    9c0a2c4cca94: Pull complete
    05f59ae6b933: Pull complete
    ec4575052267: Pull complete
    068822fb393b: Pull complete
    Digest: sha256:8338a316a476ca8d9c8f2b82b59816c21ffce1863de45150de1229c56c747d0d
    Status: Downloaded newer image for paketobuildpacks/run:full-cnb
    ===> ANALYZING
    Previous image with name "puma-sample" not found
    ===> DETECTING
    5 of 12 buildpacks participating
    paketo-buildpacks/ca-certificates 3.5.1
    paketo-buildpacks/mri             0.11.1
    paketo-buildpacks/bundler         0.7.4
    paketo-buildpacks/bundle-install  0.6.3
    paketo-buildpacks/puma            0.4.15
    ===> RESTORING
    ===> BUILDING
    
    Paketo Buildpack for CA Certificates 3.5.1
      https://github.com/paketo-buildpacks/ca-certificates
      Launch Helper: Contributing to layer
        Creating /layers/paketo-buildpacks_ca-certificates/helper/exec.d/ca-certificates-helper
    Paketo Buildpack for MRI 0.11.1
      Resolving MRI version
        Candidate version sources (in priority order):
          Gemfile   -> "3.2.1"
          <unknown> -> ""
    
        Selected MRI version (using Gemfile): 3.2.1
    
      Executing build process
        Installing MRI 3.2.1
          Completed in 16.232s
    
      Generating SBOM for /layers/paketo-buildpacks_mri/mri
          Completed in 34ms
    
      Configuring build environment
        GEM_PATH         -> "/home/cnb/.local/share/gem/ruby/3.2.0:/layers/paketo-buildpacks_mri/mri/lib/ruby/gems/3.2.0"
        MALLOC_ARENA_MAX -> "2"
    
      Configuring launch environment
        GEM_PATH         -> "/home/cnb/.local/share/gem/ruby/3.2.0:/layers/paketo-buildpacks_mri/mri/lib/ruby/gems/3.2.0"
        MALLOC_ARENA_MAX -> "2"
    
    Paketo Buildpack for Bundler 0.7.4
      Resolving Bundler version
        Candidate version sources (in priority order):
          Gemfile.lock -> "2.*.*"
          <unknown>    -> ""
    
        Selected bundler version (using Gemfile.lock): 2.4.6
    
      Executing build process
        Installing Bundler 2.4.6
          Completed in 1.763s
    
      Generating SBOM for /layers/paketo-buildpacks_bundler/bundler
          Completed in 5ms
    
      Configuring build environment
        GEM_PATH -> "$GEM_PATH:/layers/paketo-buildpacks_bundler/bundler"
    
      Configuring launch environment
        GEM_PATH -> "$GEM_PATH:/layers/paketo-buildpacks_bundler/bundler"
    
    Paketo Buildpack for Bundle Install 0.6.3
      Executing launch environment install process
        Running 'bundle config --global clean true'
        Running 'bundle config --global path /layers/paketo-buildpacks_bundle-install/launch-gems'
        Running 'bundle config --global without development:test'
        Running 'bundle config --global cache_path --parseable'
    
        Running 'bundle install'
          Fetching gem metadata from https://rubygems.org/..........
          Resolving dependencies...
    
          <snip...>

          Fetching mysql2 0.5.5
          Installing logstash-event 1.2.02
          Fetching tzinfo 1.2.11
          Installing mysql2 0.5.5 with native extensions
    
          <snip...>

          Installing puma 4.3.12 with native extensions
          Fetching rails 6.0.6.1
          Installing rails 6.0.6.1
          Fetching bootsnap 1.16.0
          Installing bootsnap 1.16.0 with native extensions
          Bundle complete! 17 Gemfile dependencies, 70 gems now installed.
          Gems in the groups 'development' and 'test' were not installed.
          Bundled gems are installed into `/layers/paketo-buildpacks_bundle-install/launch-gems`
          Completed in 1m51.974s
    
      Generating SBOM for /layers/paketo-buildpacks_bundle-install/launch-gems
          Completed in 11.999s
    
      Configuring launch environment
        BUNDLE_USER_CONFIG -> "/layers/paketo-buildpacks_bundle-install/launch-gems/config"
    
    Paketo Buildpack for Puma 0.4.15
      Assigning launch processes:
        web (default): bash -c bundle exec puma --bind tcp://0.0.0.0:${PORT:-9292}
    
    ===> EXPORTING
    Adding layer 'paketo-buildpacks/ca-certificates:helper'
    Adding layer 'paketo-buildpacks/mri:mri'
    Adding layer 'paketo-buildpacks/bundler:bundler'
    Adding layer 'paketo-buildpacks/bundle-install:launch-gems'
    Adding layer 'launch.sbom'
    Adding 1/1 app layer(s)
    Adding layer 'launcher'
    Adding layer 'config'
    Adding layer 'process-types'
    Adding label 'io.buildpacks.lifecycle.metadata'
    Adding label 'io.buildpacks.build.metadata'
    Adding label 'io.buildpacks.project.metadata'
    Setting default process type 'web'
    Saving puma-sample...
    *** Images (8236d63db653):
          puma-sample
    Adding cache layer 'paketo-buildpacks/mri:mri'
    Adding cache layer 'paketo-buildpacks/bundler:bundler'
    Adding cache layer 'cache.sbom'
    Successfully built image puma-sample
    ```
    </details>
1. image確認

    ```
    $ docker images rails-api-server
    REPOSITORY         TAG       IMAGE ID       CREATED        SIZE
    rails-api-server   latest    635f3a7771df   43 years ago   956MB
    ```

