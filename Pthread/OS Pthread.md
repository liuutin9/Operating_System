---
title: OS Pthread

---

# OS Pthread

## Implementation
```cpp
int main(int argc, char** argv) {
    assert(argc == 4);

    int n = atoi(argv[1]);
    std::string input_file_name(argv[2]);
    std::string output_file_name(argv[3]);

    TSQueue<Item*> reader_queue(READER_QUEUE_SIZE);
    TSQueue<Item*> worker_queue(WORKER_QUEUE_SIZE);
    TSQueue<Item*> writer_queue(WRITER_QUEUE_SIZE);
	Transformer transformer;
	// TODO: implements main function
	Reader* reader = new Reader(n, input_file_name, &reader_queue/*, &reader_finish*/);
	Producer* producer1 = new Producer(&reader_queue, &worker_queue, &transformer/*, &reader_finish*/);
    Producer* producer2 = new Producer(&reader_queue, &worker_queue, &transformer/*, &reader_finish*/);
    Producer* producer3 = new Producer(&reader_queue, &worker_queue, &transformer/*, &reader_finish*/);
    Producer* producer4 = new Producer(&reader_queue, &worker_queue, &transformer/*, &reader_finish*/);
	Writer* writer = new Writer(n, output_file_name, &writer_queue);
	ConsumerController* consumer_controller = new ConsumerController(
        &worker_queue, 
        &writer_queue, 
        &transformer, 
        CONSUMER_CONTROLLER_LOW_THRESHOLD_PERCENTAGE, 
        CONSUMER_CONTROLLER_HIGH_THRESHOLD_PERCENTAGE, 
        CONSUMER_CONTROLLER_CHECK_PERIOD
    );
    //std::cout << "define threads" << std::endl;
    reader->start();
    
    producer1->start();
    producer2->start();
    producer3->start();
    producer4->start();
    consumer_controller->start();
    //./main 200 ./tests/00.in ./tests/00.out

    writer->start();
    //std::cout << "start threads" << std::endl;
	reader->join();
    //std::cout << "reader finish" << std::endl;
    // producer1->join();
    // std::cout << "producer1 finish" << std::endl;
    // producer2->join();
    // producer3->join();
    // producer4->join();
    // consumer_controller->join();
    //./scripts/verify--output ./tests/00.out--answer ./tests/00.ans
    //std::cout << "consumer_controller finish" << std::endl;
    writer->join();
    //std::cout << "writer finish" << std::endl;
    //std::cout << "threads finish" << std::endl;
    // Clean up resources
    delete reader;
    delete producer1;
    delete producer2;
    delete producer3;
    delete producer4;
    delete consumer_controller;
    delete writer;

	return 0;
}

```
### 

### producer
```cpp
void Producer::start() {
    // TODO: starts a Producer thread
    pthread_create(&t, 0, Producer::process, (void*)this);
}

void* Producer::process(void* arg) {
    // TODO: implements the Producer's work
    Producer* producer = (Producer*)arg;
    
    while(true) {
	Item * item = producer->input_queue->dequeue();    
	item->val = producer->transformer->producer_transform(item->opcode, item->val);
	producer->worker_queue->enqueue(item);
    }
    return nullptr;
}
```

### consumer_controller
```cpp
void ConsumerController::start() {
    // TODO: starts a ConsumerController thread
    pthread_create(&t, 0, ConsumerController::process, (void*)this);
}

void* ConsumerController::process(void* arg) {
    // Cast the argument to ConsumerController
    ConsumerController* controller = static_cast<ConsumerController*>(arg);
    while (true) {
        // Get the current size of the worker queue
        int worker_queue_size = controller->worker_queue->get_size();
        if (worker_queue_size > controller->worker_queue->buffer_size * controller->high_threshold / 100){
            // Add a new consumer
            Consumer* new_consumer = new Consumer(
                controller->worker_queue,
                controller->writer_queue,
                controller->transformer
            );

            new_consumer->start(); // Start the consumer thread
            controller->consumers.push_back(new_consumer);
            
            std::cout << "Scaling up consumers from "<< controller->consumers.size() - 1 <<  " to " << controller->consumers.size() << std::endl;
        }
        // Scale down consumers if the worker queue falls below the low threshold
        else if (worker_queue_size < controller->worker_queue->buffer_size * controller->low_threshold / 100 && controller->consumers.size() > 1) {
            // Remove the most recently added consumer
            Consumer* last_consumer = controller->consumers.back();
            last_consumer->cancel(); // Cancel the consumer thread
            last_consumer->join();   // Wait for the thread to finish
            //delete last_consumer;    // Free memory

            controller->consumers.pop_back();
            std::cout << "Scaling down consumers from "<< controller->consumers.size() + 1 <<  " to " << controller->consumers.size() << std::endl;
            
        }

        // Ensure there is always at least one consumer
        

        // Sleep for the check period
        usleep(controller->check_period);
    }
    return nullptr;
}
```
### consumer
```cpp
void Consumer::start() {
    // TODO: starts a Consumer thread
    pthread_create(&t, 0, Consumer::process, (void*)this);
}

int Consumer::cancel() {
    is_cancel = true;
    return 0;
}

void* Consumer::process(void* arg) {
    Consumer* consumer = (Consumer*)arg;
    pthread_setcanceltype(PTHREAD_CANCEL_DEFERRED, nullptr);
    while (!consumer->is_cancel) {
        // TODO: implements the Consumer's work
        pthread_setcancelstate(PTHREAD_CANCEL_DISABLE, nullptr);
        Item *item = consumer->worker_queue->dequeue();
        item->val = consumer->transformer->consumer_transform(item->opcode, item->val);
        consumer->output_queue->enqueue(item);
        pthread_setcancelstate(PTHREAD_CANCEL_ENABLE, nullptr);
    }
    delete consumer;
    return nullptr;
}
```
### writer
```cpp
void Writer::start() {
    // TODO: starts a Writer thread
    pthread_create(&t, 0, Writer::process, (void*)this);
}

void* Writer::process(void* arg) {
    // TODO: implements the Writer's work
    Writer* writer = (Writer*)arg;
    while(writer->expected_lines--){
        Item* item = writer->output_queue->dequeue();
        writer->ofs << *item; 
        delete item; 
    }
    return nullptr;
}
```
## Experiment

### Different values of CONSUMER_CONTROLLER_CHECK_PERIOD
#### Discussion
- 大 check period
    - consumer 更加貼近 item
    - 可以降低 run time

大到一定程度會造成一直在建立或刪除 consumer，不一定會減少時間。
#### Result
- **test 00**

    | Check period | Runtime |
    | :--------: | :--------: |
    | 2 times | 11.1133 |
    | 1 times | 7.25883 |
    | 1/2 times | 5.39247 |
    | 1/10 times | 3.46049 |

    ![picture 1](Pictures/0.png)
    ![picture 2](Pictures/1.png)
    ![picture 3](Pictures/2.png)
    ![picture 4](Pictures/3.png)
    
- **test 01**

    | Check period | Runtime |
    | :--------: | :--------: |
    | 2 times | 68.8665 |
    | 1 times | 59.7507 |
    | 1/2 times | 52.8105 |
    | 1/10 times | 52.8789 |

    ![picture 1](Pictures/4.png)
    ![picture 2](Pictures/5.png)
    ![picture 3](Pictures/6.png)
    ![picture 4](Pictures/7.png)

### Different values of CONSUMER_CONTROLLER_LOW_THRESHOLD_PERCENTAGE and CONSUMER_CONTROLLER_HIGH_THRESHOLD_PERCENTAGE

#### Discussion
？？？？？
#### Result
- **test 00**

    | Threshold (min, max) | Runtime |
    | :--------: | :-----: |
    | (40%, 60%) | 6.17829 |
    | (20%, 80%) | 7.25883 |
    | (20%, 60%) | 5.77848 |
    | (40%, 80%) | 8.25213 |

    ![picture 1](Pictures/8.png)
    ![picture 2](Pictures/9.png)
    ![picture 3](Pictures/10.png)
    ![picture 4](Pictures/11.png)
    
- **test 01**

    | Threshold (min, max) | Runtime |
    | :--------: | :-----: |
    | (40%, 60%) | 58.9171 |
    | (20%, 80%) | 59.7507 |
    | (20%, 60%) | 58.7007 |
    | (40%, 80%) | 59.7340 |

    ![picture 1](Pictures/12.png)
    ![picture 2](Pictures/13.png)
    ![picture 3](Pictures/14.png)
    ![picture 4](Pictures/15.png)

 
### Different values of WORKER_QUEUE_SIZE

#### Discussion
？？？？？
#### Result
- **test 00**

    | Work queue size | Runtime |
    | :--------: | :-----: |
    | 2 times | Infinity (never end) |
    | 1 times | 7.25883 |
    | 1/2 times | 4.77562 |

    ![picture 1](Pictures/16.png)
    ![picture 2](Pictures/17.png)
    
- **test 01**

    | Work queue size | Runtime |
    | :--------: | :-----: |
    | 2 times | 60.4075 |
    | 1 times | 59.7507 |
    | 1/2 times | 56.7949 |

    ![picture 1](Pictures/18.png)
    ![picture 2](Pictures/19.png)
    ![picture 3](Pictures/20.png)

### What happens if WRITER_QUEUE_SIZE is very small?

#### Discussion
？？？？？
#### Result
![picture 1](Pictures/21.png)

### What happens if READER_QUEUE_SIZE is very small?

#### Discussion
？？？？？
#### Result
![picture 1](Pictures/22.png)