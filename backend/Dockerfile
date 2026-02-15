FROM maven:3.8.3-openjdk-17 AS build

WORKDIR /opt

COPY . .

RUN mvn clean package -DskipTests

FROM openjdk:17.0.2-jdk

COPY --from=build /opt/target/student-registration-backend-0.0.1-SNAPSHOT.jar /opt/studentapp.jar

EXPOSE 8080

CMD ["java", "-jar", "/opt/studentapp.jar"]