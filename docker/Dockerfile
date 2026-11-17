# SPDX-License-Identifier: MulanPSL-2.0
# explore_rbnx — frontier-based exploration skill image.
#
# Lightweight ROS Humble base with rclpy + numpy/scipy for grid math.
# No SLAM packages here — this skill consumes mapping's output via
# atlas-resolved /map and drives via the navigation service's gRPC.
# --network host + --ipc=host at runtime so DDS reaches /map and the
# nav service is reachable on 127.0.0.1.
ARG ROS_BASE_IMAGE=robonix-ros:jazzy-ros-base
FROM ${ROS_BASE_IMAGE}

# ── TUNA mirrors ───────────────────────────────────────────────────
# Swap apt sources (Ubuntu/ports + ROS 2) to TUNA. Use `find` + sed
# --follow-symlinks because /etc/apt/sources.list.d/ros2.sources is
# a symlink into /usr/share/ros-apt-source/ in the upstream ros image.
RUN set -eux; \
    find /etc/apt/sources.list.d/ -maxdepth 1 \( -name '*.list' -o -name '*.sources' \) -print0 \
      | xargs -0 -r sed -i --follow-symlinks \
          -e 's|http://packages\.ros\.org/ros2/ubuntu|https://mirrors.tuna.tsinghua.edu.cn/ros2/ubuntu|g' \
          -e 's|^Types: deb deb-src$|Types: deb|'; \
    if [ -f /etc/apt/sources.list ]; then \
        sed -i --follow-symlinks \
            -e 's|http://archive\.ubuntu\.com/ubuntu/\?|https://mirrors.tuna.tsinghua.edu.cn/ubuntu/|g' \
            -e 's|http://security\.ubuntu\.com/ubuntu/\?|https://mirrors.tuna.tsinghua.edu.cn/ubuntu/|g' \
            -e 's|http://ports\.ubuntu\.com/ubuntu-ports/\?|https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/|g' \
            /etc/apt/sources.list; \
    fi
ENV PIP_INDEX_URL=https://pypi.tuna.tsinghua.edu.cn/simple \
    UV_INDEX_URL=https://pypi.tuna.tsinghua.edu.cn/simple \
    HF_ENDPOINT=https://hf-mirror.com

ENV DEBIAN_FRONTEND=noninteractive \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

# Runtime deps:
#   tf2 — pose lookup (we may need map→base_link to know where we are
#         relative to a frontier candidate)
#   numpy/scipy — grid + clustering math (frontier + DBSCAN)
RUN apt-get update \
 && apt-get install -y --no-install-recommends \
        python3-pip \
        python3-numpy \
        python3-scipy \
        python3-yaml \
        ros-jazzy-tf2-ros \
        ros-jazzy-tf-transformations \
        ros-jazzy-rmw-zenoh-cpp \
 && rm -rf /var/lib/apt/lists/*

# Python deps:
#   grpcio/protobuf — atlas client
#   MCP 1.x         — robonix-api still imports mcp.server.fastmcp.FastMCP
#   fastmcp/uvicorn — MCP client/server support for the 3 skill tools
# No `pip install --upgrade pip` on noble: pip cannot uninstall the
# Debian-owned apt pip (no RECORD: "Cannot uninstall pip 24.0") and that is a
# guaranteed build failure. noble's pip 24.0 is sufficient.
# numpy<2: tf2_py (this image's ROS python C extension) is built against the
# apt numpy 1.26 ABI; an unpinned pip numpy 2.x shadows it on sys.path.
ENV PIP_BREAK_SYSTEM_PACKAGES=1
RUN python3 -m pip install --no-cache-dir \
        'grpcio==1.80.0' \
        'grpcio-tools==1.76.0' \
        'protobuf==6.33.6' \
        'pyyaml>=6.0' \
        'numpy<2' \
        'mcp>=1.28.1,<2' \
        'fastmcp>=3,<4' \
        uvicorn

WORKDIR /explore
COPY entrypoint.sh /entrypoint.sh
COPY no_shm_profile.xml /etc/fastrtps_no_shm.xml
RUN chmod +x /entrypoint.sh

# Left for operators who explicitly override RMW back to FastRTPS.
ENV FASTRTPS_DEFAULT_PROFILES_FILE=/etc/fastrtps_no_shm.xml
ENV RMW_IMPLEMENTATION=rmw_zenoh_cpp

ENTRYPOINT ["/entrypoint.sh"]
