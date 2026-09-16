# This option should only be used with decoupled projects. More details, visit
# http://www.gradle.org/docs/current/userguide/multi_project_builds.html#sec:decoupled_projects
# org.gradle.parallel=true
org.gradle.daemon=true
org.gradle.parallel=true
# Giới hạn RAM tối đa cho Gradle là 2GB, khởi điểm 512MB
org.gradle.jvmargs=-Xmx6g -XX:+UseG1GC -Dfile.encoding=UTF-8 --add-exports jdk.compiler/com.sun.tools.javac.tree=ALL-UNNAMED

# Tự động giải phóng RAM và tắt Gradle sau 1 phút không dùng (Mặc định là 3 giờ)
org.gradle.daemon.maxidle-time=60000

# Bật bộ nhớ đệm trên ổ cứng để không phải biên dịch lại từ đầu, giảm tải cho CPU/RAM
org.gradle.caching=true

# Tắt tính năng tự động tải và cấu hình lại project khi có thay đổi nhỏ
org.gradle.configureondemand=false

kotlin.daemon.jvmargs=-Xmx4096m -Dfile.encoding=UTF-8
android.useAndroidX=true
android.enableJetifier=true
# The option 'android.enableR8' is deprecated.
# The current default is 'true'.
#android.enableR8=true
android.jetifier.ignorelist = android-104.5112.05.jar, SDKFull_live_release_220524_1523.aar

#android.useAndroidX=true
#android.enableJetifier=fa
android.nonTransitiveRClass=true
kotlin.jvm.target=17
kotlin.incremental=true
kapt.incremental.apt=true
kotlin.incremental.js=false
kotlin.incremental.multiplatform=false
