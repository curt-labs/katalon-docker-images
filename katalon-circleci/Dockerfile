FROM katalonstudio/katalon-circleci

# common environment variables
ENV KATALON_KATALON_CIRCLECI_ROOT_DIR=$KATALON_ROOT_DIR/katalon-circleci
ENV KATALON_OPTS='-retry=0 -retryFailedTestCases=true -testSuitePath="Test Suites/Single Product Test Suite" -executionProfile="uat-no-price-check" -browserType="Chrome"'

# copy scripts
RUN mkdir -p $KATALON_KATALON_CIRCLECI_ROOT_DIR
WORKDIR $KATALON_KATALON_CIRCLECI_ROOT_DIR
ADD ./src ./
ADD ./source /katalon/katalon/source

RUN $KATALON_MAKE_EXECUTABLE_SCRIPT .

RUN ./index.sh
RUN $KATALON_CLEAN_UP_SCRIPT

WORKDIR /

ENTRYPOINT ls -la $KATALON_KATALON_ROOT_DIR && cat $KATALON_VERSION_FILE && $KATALON_KATALON_ROOT_DIR/scripts/katalon-execute.sh

