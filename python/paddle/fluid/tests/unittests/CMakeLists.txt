file(GLOB TEST_OPS RELATIVE "${CMAKE_CURRENT_SOURCE_DIR}" "test_*.py")
string(REPLACE ".py" "" TEST_OPS "${TEST_OPS}")

# The MKLDNN tests are skiped when the MKLDNN flag is OFF
if(NOT WITH_MKLDNN)
    foreach(src ${TEST_OPS})
        if(${src} MATCHES ".*_mkldnn_op$")
            list(REMOVE_ITEM TEST_OPS ${src})
        endif()
    endforeach()
endif(NOT WITH_MKLDNN)

if(NOT WITH_DISTRIBUTE)
    list(REMOVE_ITEM TEST_OPS test_recv_op)
    list(REMOVE_ITEM TEST_OPS test_dist_transpiler)
    list(REMOVE_ITEM TEST_OPS test_simple_dist_transpiler)
    list(REMOVE_ITEM TEST_OPS test_listen_and_serv_op)
    LIST(REMOVE_ITEM TEST_OPS test_dist_mnist)
    LIST(REMOVE_ITEM TEST_OPS test_dist_word2vec)
    LIST(REMOVE_ITEM TEST_OPS test_dist_ctr)
    LIST(REMOVE_ITEM TEST_OPS test_dist_simnet_bow)
    LIST(REMOVE_ITEM TEST_OPS test_dist_mnist_batch_merge)
    LIST(REMOVE_ITEM TEST_OPS test_dist_text_classification)
endif(NOT WITH_DISTRIBUTE)

if (${CUDNN_MAJOR_VERSION} VERSION_LESS 7)
    LIST(REMOVE_ITEM TEST_OPS test_conv2d_fusion_op)
endif()

list(REMOVE_ITEM TEST_OPS test_seq_concat_op) # FIXME(helin): https://github.com/PaddlePaddle/Paddle/issues/8290
list(REMOVE_ITEM TEST_OPS test_modified_huber_loss_op) # FIXME(qijun) https://github.com/PaddlePaddle/Paddle/issues/5184
list(REMOVE_ITEM TEST_OPS test_lstm_unit_op) # # FIXME(qijun) https://github.com/PaddlePaddle/Paddle/issues/5185
list(REMOVE_ITEM TEST_OPS test_nce) # FIXME(qijun) https://github.com/PaddlePaddle/Paddle/issues/7778
list(REMOVE_ITEM TEST_OPS test_recurrent_op) # FIXME(qijun) https://github.com/PaddlePaddle/Paddle/issues/6152
list(REMOVE_ITEM TEST_OPS test_cond_op) # FIXME(qijun): https://github.com/PaddlePaddle/Paddle/issues/5101#issuecomment-339814957

list(REMOVE_ITEM TEST_OPS op_test) # op_test is a helper python file, not a test
list(REMOVE_ITEM TEST_OPS decorators) # decorators is a helper python file, not a test
if(APPLE)
    if(NOT WITH_DISTRIBUTE)
        list(REMOVE_ITEM TEST_OPS test_desc_clone)
        list(REMOVE_ITEM TEST_OPS test_program_code)
    endif(NOT WITH_DISTRIBUTE)
    message(WARNING "These tests has been disabled in OSX before being fixed: \n test_fuse_elewise_add_act_pass \n test_detection_map_op \n test_dist_se_resnext")
    # this op is not support on mac
    list(REMOVE_ITEM TEST_OPS test_fusion_seqexpand_concat_fc_op)
    # TODO: add the unitest back when it fixed
    list(REMOVE_ITEM TEST_OPS test_detection_map_op)
    list(REMOVE_ITEM TEST_OPS test_dist_se_resnext)
    list(REMOVE_ITEM TEST_OPS test_fuse_elewise_add_act_pass)
endif()
if(NOT WITH_MKLML)
    # this op is not support on openblas
    list(REMOVE_ITEM TEST_OPS test_fusion_seqexpand_concat_fc_op)
endif()

function(py_test_modules TARGET_NAME)
  if(WITH_TESTING)
    set(options SERIAL)
    set(oneValueArgs "")
    set(multiValueArgs MODULES DEPS ENVS)
    cmake_parse_arguments(py_test_modules "${options}" "${oneValueArgs}" "${multiValueArgs}" ${ARGN})
    add_test(NAME ${TARGET_NAME}
             COMMAND env PYTHONPATH=${PADDLE_BINARY_DIR}/python ${py_test_modules_ENVS}
             ${PYTHON_EXECUTABLE} ${PADDLE_SOURCE_DIR}/tools/test_runner.py ${py_test_modules_MODULES}
             WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR})
    if (py_test_modules_SERIAL)
        set_property(TEST ${TARGET_NAME} PROPERTY RUN_SERIAL 1)
    endif()
    set_tests_properties(${TARGET_NAME} PROPERTIES TIMEOUT 600)
  endif()
endfunction()
list(REMOVE_ITEM TEST_OPS test_warpctc_op)
list(REMOVE_ITEM TEST_OPS test_dist_train)
list(REMOVE_ITEM TEST_OPS test_dist_transpiler)
list(REMOVE_ITEM TEST_OPS test_parallel_executor_crf)
list(REMOVE_ITEM TEST_OPS test_parallel_executor_fetch_feed)
list(REMOVE_ITEM TEST_OPS test_dist_se_resnext)
list(REMOVE_ITEM TEST_OPS test_dist_transformer)
list(REMOVE_ITEM TEST_OPS test_parallel_executor_transformer)
list(REMOVE_ITEM TEST_OPS test_image_classification_resnet)
foreach(TEST_OP ${TEST_OPS})
    py_test_modules(${TEST_OP} MODULES ${TEST_OP})
endforeach(TEST_OP)
py_test_modules(test_warpctc_op MODULES test_warpctc_op ENVS FLAGS_warpctc_dir=${WARPCTC_LIB_DIR} SERIAL)
if(WITH_DISTRIBUTE)
    py_test_modules(test_dist_train MODULES test_dist_train SERIAL)
    set_tests_properties(test_listen_and_serv_op PROPERTIES TIMEOUT 20)
    if(NOT APPLE)
        set_tests_properties(test_dist_mnist PROPERTIES TIMEOUT 200)
        set_tests_properties(test_dist_word2vec PROPERTIES TIMEOUT 200)
        py_test_modules(test_dist_se_resnext MODULES test_dist_se_resnext)
        set_tests_properties(test_dist_se_resnext PROPERTIES TIMEOUT 1000)
        # FIXME(typhoonzero): add this back
	#py_test_modules(test_dist_transformer MODULES test_dist_transformer)
	#set_tests_properties(test_dist_transformer PROPERTIES TIMEOUT 1000)
        # TODO(typhoonzero): make dist test parallel when fix port management issue
        set_tests_properties(test_dist_mnist test_dist_word2vec test_dist_se_resnext test_dist_ctr test_dist_simnet_bow test_dist_save_load test_dist_text_classification test_dist_mnist_batch_merge PROPERTIES RUN_SERIAL TRUE)
    endif(NOT APPLE)
    py_test_modules(test_dist_transpiler MODULES test_dist_transpiler)
endif()
py_test_modules(test_parallel_executor_crf MODULES test_parallel_executor_crf SERIAL)
py_test_modules(test_parallel_executor_fetch_feed MODULES test_parallel_executor_fetch_feed SERIAL)
set_tests_properties(test_parallel_executor_fetch_feed PROPERTIES TIMEOUT 150)
py_test_modules(test_parallel_executor_transformer MODULES test_parallel_executor_transformer SERIAL)
if(NOT APPLE)
    py_test_modules(test_image_classification_resnet MODULES test_image_classification_resnet SERIAL)
endif()
