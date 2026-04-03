cmake_minimum_required(VERSION 3.22)
project(robot_stream)

find_package(ament_cmake REQUIRED)

# Install Python server
install(PROGRAMS server.py
  DESTINATION lib/${PROJECT_NAME}
)

# Install static web files
install(DIRECTORY static/
  DESTINATION share/${PROJECT_NAME}/static
)

# Install launch files
install(DIRECTORY launch/
  DESTINATION share/${PROJECT_NAME}/launch
)

ament_package()
