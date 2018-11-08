FROM katalonstudio/katalon-circleci

# common environment variables
ENV KATALON_KATALON_CIRCLECI_ROOT_DIR=$KATALON_ROOT_DIR/katalon-circleci

# copy scripts
RUN mkdir -p $KATALON_KATALON_CIRCLECI_ROOT_DIR
WORKDIR $KATALON_KATALON_CIRCLECI_ROOT_DIR
ADD ./src ./

RUN $KATALON_MAKE_EXECUTABLE_SCRIPT .

RUN ./index.sh
RUN $KATALON_CLEAN_UP_SCRIPT

WORKDIR /

# VNC: Install, create script, delay launch until xvfb starts, make it the new entrypoint
RUN apt-get update && apt-get install -y x11vnc
EXPOSE 5999
ENV DISPLAY :99
RUN echo "#!/usr/bin/env bash" >> $KATALON_KATALON_ROOT_DIR/scripts/vnc-execute.sh
RUN echo "(sleep 5 && x11vnc -nopw -display :99 -N -forever)&" >> $KATALON_KATALON_ROOT_DIR/scripts/vnc-execute.sh
RUN echo "cat $KATALON_VERSION_FILE && $KATALON_KATALON_ROOT_DIR/scripts/katalon-execute.sh" >> $KATALON_KATALON_ROOT_DIR/scripts/vnc-execute.sh
RUN chmod +x $KATALON_KATALON_ROOT_DIR/scripts/vnc-execute.sh

# Set up test configuration
ENV KATALON_OPTS='-retry=0 -retryFailedTestCases=true -consoleLog -testSuitePath="Test Suites/pickFiveRandomTests" -executionProfile="uat-no-price-check" -browserType="customchrome"'

# Copy the test scripts into the container
ADD ./source /katalon/katalon/source

ENTRYPOINT $KATALON_KATALON_ROOT_DIR/scripts/vnc-execute.sh

