# Команды для организации кафки в неймспейсе
```bash
kubectl apply -f kafka-kraft-with-resources.yaml
kubectl get pods -w  # подождать пока все не будет Running
kubectl apply -f kafka-topics.yaml
kubectl apply -f kafka-user.yaml 
```

# Для тестирования
```bash
PRODUCER_PASS=$(kubectl get secret producer-user -n team5-ns -o jsonpath='{.data.password}' | base64 -d)
ML_PASS=$(kubectl get secret ml-user -n team5-ns -o jsonpath='{.data.password}' | base64 -d)
NOTIFICATION_PASS=$(kubectl get secret notification-user -n team5-ns -o jsonpath='{.data.password}' | base64 -d)
echo "Producer Pass: $PRODUCER_PASS"
echo "ML Pass: $ML_PASS"
echo "Notification Pass: $NOTIFICATION_PASS"
# берем пароли

kubectl apply -f kafka-client-pod.yaml

kubectl exec -it kafka-client -n team5-ns -- bash
cd /tmp
cat <<EOF > producer.properties
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required     username="producer-user"     password="<Producer Pass HERE>";
EOF
cat <<EOF > ml.properties
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
    username="ml-user" \
    password="<ML Pass HERE>";
EOF
cat <<EOF > notification.properties
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
    username="notification-user" \
    password="<Notification Pass HERE>";
EOF

/opt/kafka/bin/kafka-console-producer.sh \
  --bootstrap-server my-cluster-kafka-bootstrap:9092 \
  --topic raw-reviews \
  --producer.config producer.properties
# ждем исполнения, вводим сообщение `финальный тест, шаг 1`, выходим

/opt/kafka/bin/kafka-console-producer.sh \
  --bootstrap-server my-cluster-kafka-bootstrap:9092 \
  --topic processed-reviews \
  --producer.config ml.properties
# ждем исполнения, вводим сообщение `обработанное сообщение, шаг 2`, выходим

/opt/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server my-cluster-kafka-bootstrap:9092 \
  --topic processed-reviews \
  --consumer.config notification.properties \
  --group notification-group \
  --from-beginning
# ждем сообщения, если пришло - все гуд
```

Жизнь удалась