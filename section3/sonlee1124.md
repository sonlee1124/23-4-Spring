1. 새로운 할인 정책 개발
- 10프로 할인 구현체
public class RateDiscountPolicy implements DiscountPolicy{

    private int discountPercent = 10;
    @Override
    public int discount(Member member, int price) {
        if(member.getGrade() == Grade.VIP){
            return price * discountPercent / 100;
        }else{
            return 0;
        }
    }
}
-JUnit으로 테스트(할인된 금액이 잘 나온다 vip_o : 9000, vip_x : 0)
class RateDiscountPolicyTest {
    RateDiscountPolicy rateDiscountPolicy = new RateDiscountPolicy();

    @Test
    @DisplayName("VIP는 10프로 할일이 적용되어야 한다.")
    void vip_o(){
        //given
        Member member = new Member(1L, "memberVIP", Grade.VIP);
        //when
        int discount = rateDiscountPolicy.discount(member, 10000);
        //then
        System.out.println("discount = " + discount);
        assertThat(discount).isEqualTo(1000);
    }

    @Test
    @DisplayName("VIP가 아니면 할인이 적용되지 않아야 한다.")
    void vip_x(){
        //given
        Member member = new Member(1L, "memberVIP", Grade.BASIC);
        //when
        int discount = rateDiscountPolicy.discount(member, 10000);
        //then
        System.out.println("discount = " + discount);
        assertThat(discount).isEqualTo(0);
    }
}
할인 정책을 구현하고 테스트까지 완료 되었다.

2. 새로운 할인 정책 적용과 문제점
할인 정책을 변경하려면 클라이언트인 OrderServiceImpl코드를 수정해야한다.
//private final DiscountPolicy discountPolicy = new FixDiscountPolicy();    <- 수정 전
   	private final DiscountPolicy discountPolicy = new RateDiscountPolicy(); <- 수정 된 것

하지만 현재 코드는 OrderServiceImpl이 DiscountPolicy도 의존하는 상태이고 RateDiscountPolicy도 의존하는 상태이다.
(DIP위반)

	private DiscountPolicy discountPolicy <- 수정
	인터페이스에만 의존하도록 설계 했음, 근데 구현체가 없는데 어케 실행함?
	실행해보면 nullpointException에러가 발생됨 (null값은 참조가 불가능)
	해결 방안은 누군가가 클라이언트OrderServiceImpl에 DiscountPolicy의 구현 객체를 대신 생성하고 주입해야함

3. 관심사의 분리
지금 상황은 배우가 연극하기 위해 상대 배우도 캐스팅을 해야하는 상황
다양한 책임이 생겨 역할이 많아진다 > 현재까지 짰던 코드의 상황
관심사를 분리하자!
배우는 연기만! 섭외는 공연 기획자가!
각자의 역할에 충실하기
AppConfig등장
전체 동작 방식을 구성(설정)하기 위해 "구현 객체를 생성" 하고 "연결" 하는 책임을 가지는 별도의 설정 클래스를 생성

생성자 주입
private final MemberRepository memberRepository = new MemoryMemberRepository();
- 현재 MemberService안에서 MemoryMemberRepository()까지 넣는다? 배우가 상대 배우까지 섭외하는 방법이다.
수정 후 private final MemberRepository memberRepository;

public MemberServiceImpl(MemberRepository memberRepository) {
    this.memberRepository = memberRepository;
}
- 생성자를 통해 MemberRepositroy에 구현체가 어떤걸 넣을 지 여기서 설정해줘야함 

public MemberService memberService(){
        return new MemberServiceImpl(new MemoryMemberRepository());
    }
- 그래서 new MemberSerivceImpl()메서드 안에 new MemoryMemberRepository를 넣어줌

public class MemberServiceImpl implements MemberService{
    private final MemberRepository memberRepository;

    public MemberServiceImpl(MemberRepository memberRepository) {
        this.memberRepository = memberRepository;
    }
- 수정한게 이상태인데 여기서 MemoryMemberRepository에 대한 코드 없음
MemberServiceImpl얘는 이제 MemberRepository라는 추상화에만 의존함

AppConfig는 실제 동작에 필요한 "구현 겍체를 생성"
AppConfig는 생성한 객체 인스턴스의 참조를 "생성자를 통해 주입"해준다. 이것을 생성자 주입이라고 부른다

- OrderServiceImpl
public class OrderServiceImpl implements OrderService{

    private final MemberRepository memberRepository;
    private final DiscountPolicy discountPolicy;

    public OrderServiceImpl(MemberRepository memberRepository, DiscountPolicy discountPolicy) {
        this.memberRepository = memberRepository;
        this.discountPolicy = discountPolicy;
    }
    
- AppConfig(외부에서 주입!)
public OrderService orderService(){
        return new OrderServiceImpl(new MemoryMemberRepository(), new FixDiscountPolicy());
    }

MemberApp에서 Appconfig 사용 방법
//Appconfig를 생성 후 사용
AppConfig appConfig = new AppConfig();
MemberService memberService = appConfig.memberService();

OrderApp에서 Appconfig 사용방법
AppConfig appConfig = new AppConfig();
MemberService memberService = appConfig.memberService();
OrderService orderService = appConfig.orderService();

MemberServiceTest에서 Appconfig사용방법

MemberService memberService;
//BeforeEach : 각 테스트 실행 전에 무조건 실행
@BeforeEach
public void beforeEach(){
    AppConfig appConfig = new AppConfig();
    memberService = appConfig.memberService();
}

OrderServiceTest에서 AppConfig사용방법

MemberService memberService;
OrderService orderService;

@BeforeEach
public void BeforeEach(){
    AppConfig appConfig = new AppConfig();
    memberService = appConfig.memberService();;
    orderService = appConfig.orderService();
}

1. AppConfig를 이용해 관심사를 분리
2. 배역, 배우를 생각
3. AppConfig는 공연 기획자
4. AppConfig는 구체 클래스를 선택. 배역에 맞는 담당 배우를 선택
5. 각 배우들과 AppConfig는 본인의 역할에만 충실하면 된다.

4.AppConfig리팩터링
리팩터링이란 결과의 변경 없이 코드를 재조정하는 의미이다.
현재 AppConfig를 보면 중복이 되어 있고 역할에 따른 구현이 잘 보이지 않는 상태여서 리펙터링이 필요함

AppConfig리팩터링 후 
public class AppConfig {

    public MemberService memberService(){
        return new MemberServiceImpl(memberRepository());
    }

    private static MemberRepository memberRepository() {
        return new MemoryMemberRepository();
    }

    public OrderService orderService(){
        return new OrderServiceImpl(memberRepository(), discountPolicy());
    }

    public DiscountPolicy discountPolicy(){
        return new FixDiscountPolicy();
    }
}
new MemoryMemberRepository()를 두번 사용 했었는데 메서드로 생성하니 다른 구현체를 수정할 때 그 부분만 수정하면 된다.
이젠 엯할과 구현이 나눠진게 보인다. 전체 구성이 어떻게 되어 있는지 빠르게 파악 가능!

5. 새로운 구조와 할인 정책 적용
다시 정액 할인 정책을 정률(%) 할인 정책으로 변경해보기
FixDiscountPolicy -> RateDiscountPolicy
매우 간단함 AppConfig들어가서 FixDiscountPolicy를 RateDiscountPolicy로 수정하면 가능
public DiscountPolicy discountPolicy(){
		return new FixDiscountPolicy() -> return new RateDiscountPolicy();
    }
    
6. 전체 흐름 정리
- 새로운 할인 정책 개발 
- 새로운 할인 정책 적용과 문제점
새로 개발한 정률 할인 정책을 적용하려고 하니 클라이언트 코드인 주문 서비스 구현체도 함께 변경해야함
DiscountPolicy와 FixDiscountPolicy도 함께 의존해서 DIP위반인 상황이였다
- 관심사의 분리
다양한 책임을 가지고 있던 상황
배우는 연기만하고 공연 기획자는 섭외하는 기술만 가지고 있어야하는데 지금의 코드는 배우가 연기도하고 상대배우도 섭외해야하는 상황이였음
이때 나타난게 AppConfig가 나타남 
배우는 연기만!!! 공연 기획자는 섭외만!!! 이렇게 본인의 역할만 할 수 있게 책임이 명확해지게 만들었다
- AppConfig 리팩터링
구성 정보에서 역할과 구현을 명확하게 분리 시킴
역할이 어떻게 되어 있는지 빠르게 볼 수 있음
중복된 상황을 제거시킴

7. 좋은 객체 지향 설계의 5가지 원칙의 적용
SRP : 단일 책임 원칙
한 클래스는 하나의 책임만 가져야 한다
- SRP 단일 책임 원칙을 위해 관심사를 분리 함
- 구현 객체를 생성하고 연력하는 책임은 AppConfig가 담당

DIP : 의존 관계 역전 원칙
프로그래머는 "추상화에 의존해야함! 구체화에 의존하면 안된다." 의존성 주입은 이 원칙을 따르는 방법 중 하나다.
- 기존 코드는 OrderServiceImpl는 DIP를 지키며 DiscountPolicy 추성화 인터페이스에 의존 하는것 같았지만
FixDiscountPolicy구체화 구현 클래스에도 함께 의존한 상태였음 (DIP위반)
- AppConfig가 FixDiscountPolicy 객체 인스턴스를 클라이언트 코드 대신 생성해서 의존관계를 주입해줌

OCP : 소프트웨어 요소는 확장에는 열려 있으나 변경에는 닫혀 있어야 한다.
- 사용 영역과 구성 영역을 나눴다
- AppConfig가 의존관계를  FixDiscountPolicy를 RateDiscountPolicy로 변경해서 클라이언트 코드는 변경하지 않아도 된다.
- 소프트웨어 요소를 새롭게 확장해도 사용 영역의 변경은 닫혀 있다

8. IoC, DI, 그리고 컨테이너
IoC : 제어의 역전(Inversion of Control)
내가 호출하는게 아니라 프레임워크가 내 코드를 대신 호출해줌
- 구현 객체가 프로그램의 제어 흐름을 스스로 조종했음. 개발자 입장에선 자연스러운 흐름
- AppConfig등장 이후 구현 객체는 자신의 로직을 실행하는 역할만 담당
- 프로그램의 제어 흐름을 직접 제어하는것이 아닌 외부에서 관리하는 것을 제어의 역전(IoC)라고 한다.
프레임워크 vs 라이브러리
프레임워크 : 프레임워크가 내가 작성한 코드를 제어하고 대신 실행해주는걸 프레임워크라고 함
라이브러리 : 내가 작성한 코드가 직접 제어의 흐름을 담당하면 라이브러리

DI : 의존관계 주입
- 의존 관계는 "정적인 클래스 의존 관계와, 실행 시점에 결정되는 동적인 객체(인스턴스)의존 관계" 둘을 분리 해서 생각해야함
- 정적인 클래스 의존관계
import코드만 보고 의존 관계를 쉽게 판단이 가능함
OrderServiceImpl은 MemberReposity, DiscountPolicy에 의존하는것을 알 수 있음
그런데 이러한 클래스 의존 관계만으로 실제 어떤 객체가 OrderServiceImpl에 주입 될지는 알 수 없다.
- 동적인 객체 인스턴스 의존 관계
실행 시점에 실제 생성된 객체 인스턴스의 참조가 연결된 의존 관계
- 실행 시점에 외부에서 실제 구현 객체를 생성하고 클라이언트에 전달해서 클라이언트와 서버의 실제 의존관계가 연결 되는 것을 의존관계 주입(DI)라고 함
- 객체 인스턴스를 생성하고, 참조값을 전달해서 연결함
- 의존관계 주입을 사용하면 코드변경 안해도 된다. 클라이언트가 호출하는 대상의 타입 인스턴스를 변경 할 수 있다.
- 의존관계 주입을 사용하면 정적인 클래스를 변경하지 않고 동적인 객체 인스턴스 의존관계를 쉽게 변경 할 수 있다.

IoC컨테이너, DI컨테이너
- AppConfig처럼 객체를 생성하고 관리하면서 의존관계를 연결해주는것을 IoC컨테이너 or DI컨테이너라고함
- 의존관계 주입에 초점을 맞춰 최근엔 DI컨테이너라고도 많이 부른다
- 어셈블러, 오브젝트 팩토리 등으로 불리기도 한다.

9. 스프링으로 전환하기
- AppConfing를 스프링 기반으로 변경
@Configuration //설정 정보를 담당
public class AppConfig {
//Bean을 작성하게 되면 스프링 컨테이너에 등록이 가능해진다.
    @Bean
    public MemberService memberService(){
        return new MemberServiceImpl(memberRepository());
    }

    @Bean
    public static MemberRepository memberRepository() {
        return new MemoryMemberRepository();
    }

    @Bean
    public OrderService orderService(){
        return new OrderServiceImpl(memberRepository(), discountPolicy());
    }

    @Bean
    public DiscountPolicy discountPolicy(){
        return new RateDiscountPolicy();
    }
}
- MemberApp 수정
public static void main(String[] args) {
        //생성하게 되면 AppConfig에 있는 Bean들을 호출하여 사용이 가능해진다.
        ApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);
        MemberService memberService = applicationContext.getBean("memberService", MemberService.class);

- OrderApp 수정
public class OrderApp {
    public static void main(String[] args) {
        ApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);
        MemberService memberService = applicationContext.getBean("memberService", MemberService.class);
        OrderService orderService = applicationContext.getBean("orderService", OrderService.class);
        
스프링 컨테이너
- ApplicationContext를 스프링 컨테이너라고 함
- 기존에는 개발자가 AppConfig를 사용해 직접 객체를 생성하고 DI를 했지만, 이제부터는 스프링 컨테이너를 통해 사용
- 스프링 컨테이너는 @Configuration이 붙은 AppConfig를 구성 정보로 사용함
여기서  @Bean이라고 적힌 메서드를 모두 호출해서 반환된 객체를 스프링 컨테이너에 등록함. 이렇게 스프링 컨테이너에 등록된 객체를 스프링 빈이라고 함
- 스프링 빈은 @Bean이 붙은 메서드의 명을 스프링 빈의 이름으로 사용함. Ex)memberService, orderService
- 개발자가 필요한 객체를 AppConfig를 사용해서 직접 조회 했지만, 이제는 스프링 컨테이너를 통해 필요한 스프링 빈 객체를 찾아야함
스프링 빈은 applicationContext.getBean()메서드를 이용해 찾을 수 있다.
Ex) applicationContext.getBean("memberService", MemberService.class);
- 기존에는 개발자가 직접 자바코드로 모든 것을 했다면 이제부턴 스프링 컨테이너에 객체를 스프링 빈(@Bean)으로 등록하고 스프링 컨테이너에서 스프링 빈을 찾아서 사용하도록 변경함